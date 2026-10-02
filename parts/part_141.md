# Part 141: Lucky Forms and Validation - Forms และ Validation ใน Lucky

## บทนำ

Lucky มีระบบ type-safe forms ผ่าน SaveOperations ที่รวม form handling, validation และ database operations เข้าด้วยกัน

## SaveOperation พื้นฐาน

```crystal
# src/operations/save_user.cr
class SaveUser < User::SaveOperation
  # กำหนด columns ที่ผู้ใช้ส่งได้
  permit_columns email, name, bio, role
  
  # Attributes เพิ่มเติม (ไม่บันทึกลง database)
  attribute password : String
  attribute password_confirmation : String
  attribute terms_accepted : Bool
  
  # Validations
  before_save do
    validate_required email, name, password
    validate_uniqueness_of email
    validate_email_format
    validate_password_strength
    validate_terms
    hash_password
  end
  
  private def validate_email_format
    if val = email.value
      unless val.matches?(/.+@.+\..+/)
        email.add_error "รูปแบบอีเมลไม่ถูกต้อง"
      end
    end
  end
  
  private def validate_password_strength
    if pwd = password.value
      if pwd.size < 8
        password.add_error "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร"
      end
      
      unless pwd == password_confirmation.value
        password_confirmation.add_error "รหัสผ่านไม่ตรงกัน"
      end
      
      unless pwd.matches?(/[A-Z]/)
        password.add_error "ต้องมีตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัว"
      end
    end
  end
  
  private def validate_terms
    unless terms_accepted.value == true
      terms_accepted.add_error "กรุณายอมรับข้อกำหนดการใช้บริการ"
    end
  end
  
  private def hash_password
    if (pwd = password.value) && password.valid?
      hashed_password.value = Crypto::Bcrypt::Password.create(pwd).to_s
    end
  end
end
```

## Built-in Validators

```crystal
class SaveProduct < Product::SaveOperation
  permit_columns name, description, price, stock, category_id, active
  
  before_save do
    # Presence validation
    validate_required name, price, stock
    
    # Length validation
    validate_minimum_length name, 3
    validate_maximum_length name, 200
    validate_size_of description, min: 10, max: 5000
    
    # Numeric validation
    validate_inclusion_of category_id, in: [1, 2, 3, 4, 5].map(&.to_i64)
    
    # Uniqueness
    validate_uniqueness_of name, query: ProductQuery.new.by_category(category_id.value || 0_i64)
    
    # Format
    validate_format_of name, with: /\A[\w\s\-]+\z/
    
    # Custom validations
    validate_price
    validate_stock
  end
  
  private def validate_price
    if val = price.value
      if val < 0
        price.add_error "ราคาต้องไม่ต่ำกว่า 0"
      end
      if val > 1_000_000.0
        price.add_error "ราคาสูงเกินไป"
      end
    end
  end
  
  private def validate_stock
    if val = stock.value
      if val < 0
        stock.add_error "จำนวนสต็อกต้องไม่ต่ำกว่า 0"
      end
    end
  end
end
```

## Form Helpers ใน Pages

```crystal
# src/pages/users/new_page.cr
class Users::NewPage < MainLayout
  needs operation : SaveUser
  
  def content
    h1 "สร้างบัญชีใหม่"
    
    # form_for ใช้ path จาก Action
    form_for Users::Create do
      # Text input
      div class: "form-group" do
        label "ชื่อ", for: operation.name_param
        text_input operation.name,
          class: "form-control #{"is-invalid" unless operation.name.valid?}",
          placeholder: "ชื่อ-นามสกุล",
          autofocus: "true"
        error_for operation.name, class: "invalid-feedback"
      end
      
      # Email input
      div class: "form-group" do
        label "อีเมล", for: operation.email_param
        email_input operation.email, class: "form-control"
        error_for operation.email
      end
      
      # Password input
      div class: "form-group" do
        label "รหัสผ่าน", for: operation.password_param
        password_input operation.password, class: "form-control"
        error_for operation.password
        small "อย่างน้อย 8 ตัวอักษร มีตัวพิมพ์ใหญ่", class: "form-text text-muted"
      end
      
      # Password confirmation
      div class: "form-group" do
        label "ยืนยันรหัสผ่าน", for: operation.password_confirmation_param
        password_input operation.password_confirmation, class: "form-control"
        error_for operation.password_confirmation
      end
      
      # Bio (textarea)
      div class: "form-group" do
        label "ประวัติย่อ (ถ้ามี)", for: operation.bio_param
        textarea_input operation.bio,
          class: "form-control",
          rows: "4",
          placeholder: "เล่าเรื่องราวของคุณ..."
      end
      
      # Checkbox
      div class: "form-check" do
        checkbox_input operation.terms_accepted, class: "form-check-input"
        label "ฉันยอมรับข้อกำหนดการใช้บริการ", for: operation.terms_accepted_param, class: "form-check-label"
        error_for operation.terms_accepted
      end
      
      # Submit
      button type: "submit", class: "btn btn-primary" do
        text "สร้างบัญชี"
      end
      
      link "ยกเลิก", to: Home::Index, class: "btn btn-secondary"
    end
  end
end
```

## Select Inputs

```crystal
class Products::EditPage < MainLayout
  needs operation : SaveProduct
  needs categories : CategoryQuery::SelectResult
  
  def content
    form_for Products::Update.with(product_id: operation.record.try(&.id) || 0_i64) do
      # Select from array
      div class: "form-group" do
        label "หมวดหมู่"
        select_input operation.category_id do
          option "-- เลือกหมวดหมู่ --", value: ""
          categories.each do |cat|
            option cat.name,
              value: cat.id.to_s,
              selected: operation.category_id.value == cat.id
          end
        end
        error_for operation.category_id
      end
      
      # Select with named options
      div class: "form-group" do
        label "สถานะ"
        select_input operation.status,
          options: [
            {"กำลังขาย" => "active"},
            {"หยุดขาย" => "inactive"},
            {"หมดสต็อก" => "out_of_stock"},
          ]
      end
      
      # Radio buttons
      div class: "form-group" do
        label "ระดับราคา"
        ["budget", "mid", "premium"].each do |tier|
          div class: "form-check" do
            radio_input operation.price_tier,
              value: tier,
              class: "form-check-input"
            label tier.capitalize, class: "form-check-label"
          end
        end
      end
      
      submit "บันทึก"
    end
  end
end
```

## Update Operations

```crystal
class UpdateUser < User::SaveOperation
  permit_columns name, bio, avatar_url
  
  # Password update แยกต่างหาก
  attribute current_password : String
  attribute new_password : String
  attribute new_password_confirmation : String
  
  before_save do
    validate_required name
    validate_maximum_length name, 100
    
    # Password change (optional)
    if new_password.value.presence
      verify_current_password
      validate_new_password
    end
  end
  
  private def verify_current_password
    user = record
    return unless user
    
    if current = current_password.value
      unless Crypto::Bcrypt::Password.new(user.hashed_password).verify(current)
        current_password.add_error "รหัสผ่านปัจจุบันไม่ถูกต้อง"
      end
    else
      current_password.add_error "กรุณาใส่รหัสผ่านปัจจุบัน"
    end
  end
  
  private def validate_new_password
    pwd = new_password.value
    
    if pwd.nil? || pwd.empty?
      new_password.add_error "กรุณาใส่รหัสผ่านใหม่"
      return
    end
    
    if pwd.size < 8
      new_password.add_error "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร"
    end
    
    if pwd != new_password_confirmation.value
      new_password_confirmation.add_error "รหัสผ่านไม่ตรงกัน"
    end
    
    if new_password.valid?
      hashed_password.value = Crypto::Bcrypt::Password.create(pwd).to_s
    end
  end
end
```

## File Upload

```crystal
class SaveUserAvatar < User::SaveOperation
  attribute avatar : Lucky::UploadedFile
  
  before_save do
    process_avatar
  end
  
  private def process_avatar
    if uploaded = avatar.value
      # Validate type
      unless ["image/jpeg", "image/png", "image/gif", "image/webp"].includes?(uploaded.content_type)
        avatar.add_error "รองรับเฉพาะ JPG, PNG, GIF, WebP"
        return
      end
      
      # Validate size (max 5MB)
      if uploaded.size > 5 * 1024 * 1024
        avatar.add_error "ขนาดไฟล์ต้องไม่เกิน 5MB"
        return
      end
      
      # Save file
      filename = "#{Random::Secure.hex(16)}#{File.extname(uploaded.filename)}"
      dest = "public/uploads/avatars/#{filename}"
      Dir.mkdir_p(File.dirname(dest))
      File.copy(uploaded.tempfile.path, dest)
      
      avatar_url.value = "/uploads/avatars/#{filename}"
    end
  end
end
```

## Error Display

```crystal
# Helpers สำหรับ display errors
module FormHelpers
  def error_for(attribute : Avram::Attribute, **html_options)
    if attribute.errors.any?
      div class: "invalid-feedback", **html_options do
        attribute.errors.each { |err| p err }
      end
    end
  end
  
  def alert_if_errors(operation : Avram::Operation)
    if operation.errors.any?
      div class: "alert alert-danger" do
        h5 "กรุณาแก้ไขข้อผิดพลาดต่อไปนี้:"
        ul do
          operation.errors.each do |field, messages|
            messages.each { |msg| li "#{field}: #{msg}" }
          end
        end
      end
    end
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Contact Form

```crystal
# Operation ไม่บันทึก database
class SendContactMessage < Avram::Operation
  attribute name : String
  attribute email : String
  attribute subject : String
  attribute message : String
  attribute phone : String?
  
  before_submit do
    validate_required name, email, subject, message
    validate_format_of email, with: /.+@.+\..+/
    validate_minimum_length message, 20
    validate_maximum_length message, 2000
    
    if phone.value.presence
      validate_format_of phone, with: /\A[0-9\-\+\(\) ]+\z/
    end
  end
  
  def submit(&)
    run_before_submit_callbacks
    
    if valid?
      # ส่ง email
      ContactMailer.deliver(
        to: "support@example.com",
        from: email.value!,
        subject: subject.value!,
        body: build_email_body
      )
      
      yield true, self
    else
      yield false, self
    end
  end
  
  private def build_email_body : String
    <<-BODY
    จาก: #{name.value}
    อีเมล: #{email.value}
    โทรศัพท์: #{phone.value || "ไม่ระบุ"}
    
    #{message.value}
    BODY
  end
end

# Action
class Contact::Create < BrowserAction
  post "/contact" do
    op = SendContactMessage.new(params)
    
    op.submit do |success, operation|
      if success
        redirect to: Contact::Thanks, notice: "ส่งข้อความเรียบร้อยแล้ว!"
      else
        render Contact::NewPage, operation: operation
      end
    end
  end
end

# Page
class Contact::NewPage < MainLayout
  needs operation : SendContactMessage
  
  def content
    h1 "ติดต่อเรา"
    
    form_for Contact::Create do
      div class: "form-group" do
        label "ชื่อ"
        text_input operation.name, class: "form-control"
        error_for operation.name
      end
      
      div class: "form-group" do
        label "อีเมล"
        email_input operation.email, class: "form-control"
        error_for operation.email
      end
      
      div class: "form-group" do
        label "หัวข้อ"
        text_input operation.subject, class: "form-control"
        error_for operation.subject
      end
      
      div class: "form-group" do
        label "ข้อความ"
        textarea_input operation.message, class: "form-control", rows: "6"
        error_for operation.message
      end
      
      submit "ส่งข้อความ", class: "btn btn-primary"
    end
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SaveOperation**: จัดการ form submission + database
2. **permit_columns**: กำหนด allowed fields
3. **attribute**: เพิ่ม fields พิเศษ
4. **Built-in Validators**: validate_required, validate_format_of, validate_minimum_length ฯลฯ
5. **Custom Validators**: method ใน operation
6. **before_save**: callbacks ก่อนบันทึก
7. **Form Helpers**: form_for, text_input, error_for ฯลฯ
8. **Select/Radio**: dropdown และ radio button inputs
9. **File Upload**: Lucky::UploadedFile
10. **Non-DB Operations**: operations ที่ไม่บันทึก database

Lucky's form system รับประกัน type safety ตั้งแต่ HTTP params จนถึง database
