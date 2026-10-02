# Part 166: BDD Style Testing ใน Crystal

## บทนำ

BDD (Behavior-Driven Development) เป็นแนวทางพัฒนา software ที่เน้นการเขียน tests ในแบบ "ภาษาธรรมชาติ" เพื่อให้ทุกฝ่าย (developers, testers, business) เข้าใจได้ง่าย

## BDD vs TDD

```
TDD: เน้น "How it works" (implementation)
BDD: เน้น "What it does" (behavior)

TDD:
  test "add should return sum"

BDD:
  describe Calculator do
    context "เมื่อบวกเลขสองจำนวน" do
      it "คืนค่าผลรวม" do
        ...
      end
    end
  end
```

## Given/When/Then Pattern

```crystal
# Crystal Spec รองรับ Given/When/Then ผ่าน context/it
describe "User Registration" do
  describe "สมัครสมาชิกใหม่" do
    context "Given: ผู้ใช้กรอกข้อมูลถูกต้องครบถ้วน" do
      it "When: กด submit / Then: account ถูกสร้าง" do
        # Given
        valid_params = {
          email:    "alice@example.com",
          username: "alice",
          password: "Secure@123"
        }

        # When
        result = UserService.register(**valid_params)

        # Then
        result.success?.should be_true
        result.user.email.should eq("alice@example.com")
      end

      it "When: กด submit / Then: ส่ง welcome email" do
        email_spy = SpyEmailService.new
        service = UserService.new(email: email_spy)

        service.register(
          email: "alice@example.com",
          username: "alice",
          password: "Secure@123"
        )

        email_spy.welcome_sent_to?("alice@example.com").should be_true
      end
    end

    context "Given: ผู้ใช้กรอก email ไม่ถูกต้อง" do
      it "When: กด submit / Then: แสดง error message" do
        result = UserService.register(
          email:    "not-an-email",
          username: "alice",
          password: "Secure@123"
        )

        result.success?.should be_false
        result.errors[:email]?.should_not be_nil
      end
    end

    context "Given: email นี้มีในระบบแล้ว" do
      it "When: กด submit / Then: แสดง duplicate error" do
        # Setup
        UserService.register(
          email: "existing@example.com",
          username: "user1",
          password: "Secure@123"
        )

        # Try duplicate
        result = UserService.register(
          email:    "existing@example.com",
          username: "user2",
          password: "Secure@123"
        )

        result.success?.should be_false
        result.errors[:email]?.should contain("ถูกใช้แล้ว")
      end
    end
  end
end
```

## User Story Tests

```crystal
# User Story: "ในฐานะลูกค้า ฉันต้องการเพิ่มสินค้าลงตะกร้า เพื่อซื้อในภายหลัง"
describe "Shopping Cart Feature" do
  describe "เพิ่มสินค้าลงตะกร้า" do
    context "เมื่อสินค้ามีใน stock" do
      it "เพิ่มสินค้าลงตะกร้าได้" do
        # Arrange
        product = Product.new(id: 1_i64, name: "Crystal Book", price: 599.0, stock: 10)
        cart = Cart.new(user_id: 1_i64)

        # Act
        result = cart.add_item(product, quantity: 2)

        # Assert
        result.should be_truthy
        cart.items.size.should eq(1)
        cart.items.first.quantity.should eq(2)
      end

      it "อัปเดต quantity เมื่อเพิ่มสินค้าชิ้นเดิม" do
        product = Product.new(id: 1_i64, name: "Crystal Book", price: 599.0, stock: 10)
        cart = Cart.new(user_id: 1_i64)

        cart.add_item(product, quantity: 2)
        cart.add_item(product, quantity: 3)

        cart.items.size.should eq(1)
        cart.items.first.quantity.should eq(5)
      end
    end

    context "เมื่อสินค้าหมด stock" do
      it "ไม่สามารถเพิ่มสินค้าได้" do
        product = Product.new(id: 1_i64, name: "Rare Book", price: 999.0, stock: 0)
        cart = Cart.new(user_id: 1_i64)

        expect_raises(Cart::OutOfStockError) do
          cart.add_item(product, quantity: 1)
        end
      end
    end

    context "เมื่อ quantity มากกว่า stock" do
      it "เพิ่มได้แค่จำนวนที่มีใน stock" do
        product = Product.new(id: 1_i64, name: "Limited Book", price: 799.0, stock: 3)
        cart = Cart.new(user_id: 1_i64)

        expect_raises(Cart::InsufficientStockError) do
          cart.add_item(product, quantity: 5)
        end
      end
    end
  end

  describe "ลบสินค้าออกจากตะกร้า" do
    context "เมื่อสินค้าอยู่ในตะกร้า" do
      it "ลบสินค้าออกได้" do
        product = Product.new(id: 1_i64, name: "Crystal Book", price: 599.0, stock: 10)
        cart = Cart.new(user_id: 1_i64)
        cart.add_item(product, quantity: 2)

        cart.remove_item(product.id)

        cart.items.should be_empty
      end
    end

    context "เมื่อสินค้าไม่อยู่ในตะกร้า" do
      it "ไม่เกิด error" do
        cart = Cart.new(user_id: 1_i64)

        # ไม่ควร raise error
        cart.remove_item(999_i64)
        cart.items.should be_empty
      end
    end
  end

  describe "คำนวณยอดรวม" do
    it "คำนวณ total ได้ถูกต้อง" do
      cart = Cart.new(user_id: 1_i64)
      cart.add_item(Product.new(id: 1_i64, name: "Book A", price: 299.0, stock: 10), quantity: 2)
      cart.add_item(Product.new(id: 2_i64, name: "Book B", price: 499.0, stock: 5), quantity: 1)

      cart.total.should eq(1097.0)  # 299*2 + 499
    end
  end
end
```

## Spec::DSL

Crystal Spec มี DSL ที่สมบูรณ์สำหรับ BDD:

```crystal
# src/bank_account.cr
class BankAccount
  class InsufficientFunds < Exception
    def initialize(amount : Float64, balance : Float64)
      super("ยอดเงินไม่เพียงพอ: ต้องการ #{amount} มีแค่ #{balance}")
    end
  end

  class NegativeAmount < Exception
    def initialize(amount : Float64)
      super("จำนวนเงินต้องมากกว่า 0: #{amount}")
    end
  end

  getter balance : Float64
  getter owner : String
  getter transactions : Array(NamedTuple(type: String, amount: Float64, timestamp: Time))

  def initialize(@owner : String, initial_balance : Float64 = 0.0)
    raise NegativeAmount.new(initial_balance) if initial_balance < 0
    @balance = initial_balance
    @transactions = [] of NamedTuple(type: String, amount: Float64, timestamp: Time)
  end

  def deposit(amount : Float64)
    raise NegativeAmount.new(amount) if amount <= 0
    @balance += amount
    @transactions << {type: "deposit", amount: amount, timestamp: Time.utc}
    self
  end

  def withdraw(amount : Float64)
    raise NegativeAmount.new(amount) if amount <= 0
    raise InsufficientFunds.new(amount, @balance) if amount > @balance
    @balance -= amount
    @transactions << {type: "withdrawal", amount: amount, timestamp: Time.utc}
    self
  end

  def transfer_to(other : BankAccount, amount : Float64)
    withdraw(amount)
    other.deposit(amount)
    self
  end
end
```

```crystal
# spec/bank_account_spec.cr - BDD Style
require "./spec_helper"
require "../src/bank_account"

describe BankAccount do
  describe "การสร้างบัญชี" do
    context "เมื่อสร้างบัญชีใหม่" do
      it "มียอดเงินเริ่มต้นตามที่กำหนด" do
        account = BankAccount.new("Alice", 1000.0)
        account.balance.should eq(1000.0)
      end

      it "มีชื่อเจ้าของถูกต้อง" do
        account = BankAccount.new("Bob")
        account.owner.should eq("Bob")
      end

      it "ยอดเงินเริ่มต้น 0 เมื่อไม่ระบุ" do
        account = BankAccount.new("Alice")
        account.balance.should eq(0.0)
      end
    end

    context "เมื่อสร้างด้วยยอดเงินติดลบ" do
      it "raise NegativeAmount" do
        expect_raises(BankAccount::NegativeAmount) do
          BankAccount.new("Alice", -100.0)
        end
      end
    end
  end

  describe "การฝากเงิน" do
    context "เมื่อฝากเงินจำนวน valid" do
      it "ยอดเงินเพิ่มขึ้น" do
        account = BankAccount.new("Alice", 500.0)
        account.deposit(200.0)
        account.balance.should eq(700.0)
      end

      it "บันทึก transaction" do
        account = BankAccount.new("Alice")
        account.deposit(500.0)
        account.transactions.size.should eq(1)
        account.transactions.first[:type].should eq("deposit")
        account.transactions.first[:amount].should eq(500.0)
      end

      it "รองรับ method chaining" do
        account = BankAccount.new("Alice")
        account.deposit(100.0).deposit(200.0).deposit(300.0)
        account.balance.should eq(600.0)
      end
    end

    context "เมื่อฝากเงินจำนวนไม่ valid" do
      it "raise NegativeAmount สำหรับจำนวนติดลบ" do
        account = BankAccount.new("Alice", 500.0)
        expect_raises(BankAccount::NegativeAmount) do
          account.deposit(-100.0)
        end
      end

      it "raise NegativeAmount สำหรับศูนย์" do
        account = BankAccount.new("Alice", 500.0)
        expect_raises(BankAccount::NegativeAmount) do
          account.deposit(0.0)
        end
      end
    end
  end

  describe "การถอนเงิน" do
    context "เมื่อมีเงินพอ" do
      it "ยอดเงินลดลง" do
        account = BankAccount.new("Alice", 1000.0)
        account.withdraw(300.0)
        account.balance.should eq(700.0)
      end

      it "บันทึก transaction" do
        account = BankAccount.new("Alice", 1000.0)
        account.withdraw(300.0)
        account.transactions.last[:type].should eq("withdrawal")
        account.transactions.last[:amount].should eq(300.0)
      end

      it "ถอนได้ทั้งยอด" do
        account = BankAccount.new("Alice", 500.0)
        account.withdraw(500.0)
        account.balance.should eq(0.0)
      end
    end

    context "เมื่อเงินไม่พอ" do
      it "raise InsufficientFunds" do
        account = BankAccount.new("Alice", 100.0)
        error = expect_raises(BankAccount::InsufficientFunds) do
          account.withdraw(200.0)
        end
        error.message.should contain("ยอดเงินไม่เพียงพอ")
      end

      it "ยอดเงินไม่เปลี่ยนเมื่อถอนไม่สำเร็จ" do
        account = BankAccount.new("Alice", 100.0)
        begin
          account.withdraw(200.0)
        rescue BankAccount::InsufficientFunds
        end
        account.balance.should eq(100.0)
      end
    end
  end

  describe "การโอนเงิน" do
    context "เมื่อมีเงินพอ" do
      it "ยอดเงินต้นลดลง ปลายทางเพิ่มขึ้น" do
        sender   = BankAccount.new("Alice", 1000.0)
        receiver = BankAccount.new("Bob", 200.0)

        sender.transfer_to(receiver, 300.0)

        sender.balance.should eq(700.0)
        receiver.balance.should eq(500.0)
      end

      it "บันทึก transactions ทั้งสองฝั่ง" do
        sender   = BankAccount.new("Alice", 1000.0)
        receiver = BankAccount.new("Bob")

        sender.transfer_to(receiver, 500.0)

        sender.transactions.last[:type].should eq("withdrawal")
        receiver.transactions.last[:type].should eq("deposit")
      end
    end

    context "เมื่อเงินไม่พอ" do
      it "ยกเลิก transaction ทั้งหมด" do
        sender   = BankAccount.new("Alice", 100.0)
        receiver = BankAccount.new("Bob", 500.0)

        expect_raises(BankAccount::InsufficientFunds) do
          sender.transfer_to(receiver, 200.0)
        end

        # ยอดเงินต้องไม่เปลี่ยน
        sender.balance.should eq(100.0)
        receiver.balance.should eq(500.0)
      end
    end
  end
end
```

## Feature Specs (กว้างกว่า)

```crystal
# spec/features/checkout_feature_spec.cr
describe "Feature: Checkout" do
  describe "As a customer" do
    describe "I want to checkout my cart" do
      describe "So that I can receive my products" do
        context "Given I have items in my cart" do
          context "And I have a valid payment method" do
            it "Then I should be able to complete the order" do
              # Setup
              user = create_user
              product = create_product(price: 599.0, stock: 5)
              cart = create_cart(user, items: [{product: product, qty: 2}])
              payment = valid_credit_card

              # Execute
              result = CheckoutService.new.checkout(
                cart:    cart,
                user:    user,
                payment: payment
              )

              # Verify
              result.success?.should be_true
              result.order.status.should eq("confirmed")
              result.order.total.should eq(1198.0)
            end
          end

          context "And my payment is declined" do
            it "Then I should see an error message" do
              user = create_user
              product = create_product(price: 599.0, stock: 5)
              cart = create_cart(user, items: [{product: product, qty: 1}])
              declined_card = declined_payment_method

              result = CheckoutService.new.checkout(
                cart:    cart,
                user:    user,
                payment: declined_card
              )

              result.success?.should be_false
              result.error.should contain("payment declined")
            end
          end
        end

        context "Given my cart is empty" do
          it "Then I cannot checkout" do
            user = create_user
            cart = Cart.new(user_id: user.id)

            expect_raises(Cart::EmptyCartError) do
              CheckoutService.new.checkout(cart: cart, user: user, payment: valid_payment)
            end
          end
        end
      end
    end
  end
end
```

## แบบฝึกหัด

1. เขียน BDD spec สำหรับ `LoanCalculator` ที่คำนวณ เงินกู้ ดอกเบี้ย และ ค่างวด
2. สร้าง feature spec สำหรับ "User Login with 2FA" ครอบคลุมทุก scenarios
3. เขียน story-based test สำหรับ "Password Reset Flow"
4. แปลง unit tests ที่มีอยู่ให้เป็น BDD style โดยใช้ Given/When/Then

## สรุป

BDD Style Testing ใน Crystal:
- **describe/context/it**: สร้าง readable test structure
- **Given/When/Then**: แสดง scenario ชัดเจน
- **User stories**: test ที่ reflect business requirements
- **Feature specs**: ทดสอบ behavior ระดับ feature
- **Spec::DSL**: Crystal's built-in DSL รองรับ BDD ได้ดี

BDD ช่วยให้ tests เป็น documentation ที่ทุกคนในทีมอ่านเข้าใจได้
