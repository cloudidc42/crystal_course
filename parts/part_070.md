# Part 70: String Algorithms

## บทนำ

String algorithms เป็นพื้นฐานสำคัญของ computer science ในบทนี้เราจะเรียนรู้ algorithms สำหรับการค้นหา การวัดความคล้ายคลึง และการวิเคราะห์ข้อความ

---

## 1. KMP (Knuth-Morris-Pratt) Algorithm

```crystal
# KMP: ค้นหา pattern ใน text อย่างมีประสิทธิภาพ O(n + m)
# ดีกว่า naive approach O(n*m) สำหรับ patterns ที่มี repetition

def kmp_failure_function(pattern : String) : Array(Int32)
  m = pattern.size
  failure = Array.new(m, 0)
  j = 0
  i = 1
  
  while i < m
    if pattern[i] == pattern[j]
      j += 1
      failure[i] = j
      i += 1
    elsif j > 0
      j = failure[j - 1]
    else
      failure[i] = 0
      i += 1
    end
  end
  
  failure
end

def kmp_search(text : String, pattern : String) : Array(Int32)
  positions = [] of Int32
  n = text.size
  m = pattern.size
  return positions if m == 0
  
  failure = kmp_failure_function(pattern)
  j = 0  # pattern index
  
  n.times do |i|
    while j > 0 && text[i] != pattern[j]
      j = failure[j - 1]
    end
    
    if text[i] == pattern[j]
      j += 1
    end
    
    if j == m
      positions << (i - m + 1)
      j = failure[j - 1]
    end
  end
  
  positions
end

# ทดสอบ KMP
text = "AABAACAADAABAABA"
pattern = "AABA"
positions = kmp_search(text, pattern)
puts "Pattern '#{pattern}' found at positions: #{positions.inspect}"
# => [0, 9, 12]

text2 = "the cat sat on the mat near the bat"
pattern2 = "the"
puts "Pattern '#{pattern2}' found at: #{kmp_search(text2, pattern2).inspect}"
# => [0, 18, 30]

# เปรียบเทียบกับ built-in
def naive_search(text : String, pattern : String) : Array(Int32)
  positions = [] of Int32
  (0..text.size - pattern.size).each do |i|
    positions << i if text[i, pattern.size] == pattern
  end
  positions
end

# ทดสอบว่าให้ผลเหมือนกัน
require "benchmark"
test_text = "abcabcabcabc" * 100
test_pattern = "abcabc"

Benchmark.ips do |x|
  x.report("KMP") { kmp_search(test_text, test_pattern) }
  x.report("Naive") { naive_search(test_text, test_pattern) }
  x.report("Built-in scan") { test_text.scan(/#{Regex.escape(test_pattern)}/).size }
end
```

---

## 2. Boyer-Moore-Horspool Algorithm

```crystal
# Boyer-Moore-Horspool: ค้นหา pattern โดย skip characters
# เฉลี่ย O(n/m) สำหรับ random text - เร็วมากสำหรับ long patterns

def bmh_search(text : String, pattern : String) : Array(Int32)
  positions = [] of Int32
  n = text.size
  m = pattern.size
  return positions if m == 0 || m > n
  
  # Build bad character shift table
  shift = Hash(Char, Int32).new(m)
  (0...m - 1).each do |i|
    shift[pattern[i]] = m - 1 - i
  end
  
  # Search
  i = m - 1
  while i < n
    j = m - 1
    k = i
    
    while j >= 0 && text[k] == pattern[j]
      j -= 1
      k -= 1
    end
    
    if j == -1
      positions << (k + 1)
    end
    
    i += shift[text[i]]? || m
  end
  
  positions
end

# ทดสอบ
text = "HERE IS A SIMPLE EXAMPLE"
pattern = "EXAMPLE"
puts bmh_search(text, pattern).inspect  # => [17]

text2 = "ABAAABAB"
pattern2 = "ABAB"
puts bmh_search(text2, pattern2).inspect  # => [4]
```

---

## 3. Levenshtein Distance

```crystal
# Levenshtein Distance: จำนวน edit operations ขั้นต่ำ
# Operations: insert, delete, substitute

def levenshtein_distance(s1 : String, s2 : String) : Int32
  m = s1.size
  n = s2.size
  
  # Base cases
  return n if m == 0
  return m if n == 0
  
  # DP matrix
  dp = Array.new(m + 1) { |i| Array.new(n + 1, 0) }
  
  (0..m).each { |i| dp[i][0] = i }
  (0..n).each { |j| dp[0][j] = j }
  
  s1.chars.each_with_index do |c1, i|
    s2.chars.each_with_index do |c2, j|
      cost = c1 == c2 ? 0 : 1
      dp[i + 1][j + 1] = [
        dp[i][j + 1] + 1,    # deletion
        dp[i + 1][j] + 1,    # insertion
        dp[i][j] + cost,     # substitution
      ].min
    end
  end
  
  dp[m][n]
end

# ทดสอบ
puts levenshtein_distance("kitten", "sitting")    # => 3
puts levenshtein_distance("saturday", "sunday")   # => 3
puts levenshtein_distance("", "hello")            # => 5
puts levenshtein_distance("hello", "hello")       # => 0
puts levenshtein_distance("รับ", "ยาก")           # Thai

# Similarity percentage
def similarity_percentage(s1 : String, s2 : String) : Float64
  max_len = [s1.size, s2.size].max
  return 100.0 if max_len == 0
  
  dist = levenshtein_distance(s1, s2)
  (1.0 - dist.to_f / max_len) * 100.0
end

puts "%.1f%%" % similarity_percentage("hello", "helo")     # => 80.0%
puts "%.1f%%" % similarity_percentage("crystal", "Crystal") # => 85.7%

# Spell checker
def suggest_corrections(word : String, dictionary : Array(String), max_dist : Int32 = 2) : Array(String)
  dictionary
    .select { |w| levenshtein_distance(word, w) <= max_dist }
    .sort_by { |w| levenshtein_distance(word, w) }
end

dict = ["hello", "help", "world", "word", "work", "heal", "hell"]
puts suggest_corrections("helo", dict).inspect  # => ["hello", "help", "hell"]
puts suggest_corrections("wrld", dict).inspect  # => ["word", "world"]
```

---

## 4. Hamming Distance

```crystal
# Hamming Distance: ตำแหน่งที่ต่างกันระหว่าง 2 strings ที่มีความยาวเท่ากัน

def hamming_distance(s1 : String, s2 : String) : Int32
  raise ArgumentError.new("Strings must have equal length") unless s1.size == s2.size
  
  s1.chars.zip(s2.chars).count { |c1, c2| c1 != c2 }
end

puts hamming_distance("GAGCCTACTAACGGGAT", "CATCGTAATGACGGCCT")  # => 7
puts hamming_distance("karolin", "kathrin")  # => 3
puts hamming_distance("1011101", "1001001")  # => 2

# Hamming distance สำหรับ binary strings (error detection)
def bit_errors(original : String, received : String) : Int32
  hamming_distance(original, received)
end

puts bit_errors("00000", "00100")  # => 1 bit error
puts bit_errors("11111", "10101")  # => 2 bit errors

# ใช้ใน DNA analysis
def dna_similarity(seq1 : String, seq2 : String) : String
  return "Length mismatch" if seq1.size != seq2.size
  
  dist = hamming_distance(seq1, seq2)
  pct = (1.0 - dist.to_f / seq1.size) * 100
  "#{dist} differences (#{pct.round(1)}% similar)"
end

dna1 = "ATCGATCGATCG"
dna2 = "ATCGATCGTTCG"
puts dna_similarity(dna1, dna2)  # => "2 differences (83.3% similar)"
```

---

## 5. Palindrome Check

```crystal
# ตรวจสอบ palindrome แบบต่างๆ

# Simple palindrome
def palindrome?(str : String) : Bool
  str == str.reverse
end

puts palindrome?("racecar")   # => true
puts palindrome?("hello")     # => false
puts palindrome?("a")         # => true
puts palindrome?("aa")        # => true

# Case-insensitive, ignore spaces and punctuation
def palindrome_ci?(str : String) : Bool
  cleaned = str.downcase.gsub(/[^a-z0-9]/, "")
  cleaned == cleaned.reverse
end

puts palindrome_ci?("A man a plan a canal Panama")  # => true
puts palindrome_ci?("Was it a car or a cat I saw?") # => true
puts palindrome_ci?("race a car")                   # => false

# Palindrome ที่รองรับ Unicode
def unicode_palindrome?(str : String) : Bool
  chars = str.chars
  chars == chars.reverse
end

puts unicode_palindrome?("กาฬ")  # ขึ้นอยู่กับตัวอักษร

# หา palindrome substrings
def find_palindromes(str : String, min_length : Int32 = 2) : Array(String)
  result = [] of String
  n = str.size
  
  # Odd length palindromes
  (0...n).each do |center|
    length = 0
    while center - length >= 0 && center + length < n
      break unless str[center - length] == str[center + length]
      if length * 2 + 1 >= min_length
        result << str[center - length, length * 2 + 1]
      end
      length += 1
    end
  end
  
  # Even length palindromes
  (0...n - 1).each do |center|
    length = 0
    while center - length >= 0 && center + length + 1 < n
      break unless str[center - length] == str[center + length + 1]
      if (length + 1) * 2 >= min_length
        result << str[center - length, (length + 1) * 2]
      end
      length += 1
    end
  end
  
  result.uniq.sort_by { |p| [-p.size, p] }
end

puts find_palindromes("abacaba").inspect
# => ["abacaba", "abacaba", "aba", "aba", "aa", ...]

puts find_palindromes("racecar").inspect
# => ["racecar", "aceca", "cec"]
```

---

## 6. Anagram Check

```crystal
# ตรวจสอบ anagram: คำที่ใช้ตัวอักษรชุดเดียวกันแต่เรียงต่างกัน

def anagram?(s1 : String, s2 : String) : Bool
  return false if s1.size != s2.size
  s1.chars.sort == s2.chars.sort
end

puts anagram?("listen", "silent")    # => true
puts anagram?("hello", "world")      # => false
puts anagram?("triangle", "integral") # => true
puts anagram?("abc", "cba")          # => true

# Case-insensitive anagram
def anagram_ci?(s1 : String, s2 : String) : Bool
  s1.downcase.chars.sort == s2.downcase.chars.sort
end

puts anagram_ci?("Listen", "Silent")  # => true
puts anagram_ci?("Astronomer", "Moon starer")  # ต้องลบ spaces ด้วย

# Anagram with spaces
def anagram_text?(s1 : String, s2 : String) : Bool
  clean = ->(s : String) { s.downcase.delete(" ").chars.sort }
  clean.call(s1) == clean.call(s2)
end

puts anagram_text?("Astronomer", "Moon starer")   # => true
puts anagram_text?("Conversation", "Voices rant on") # => true

# Anagram groups จาก word list
def group_anagrams(words : Array(String)) : Array(Array(String))
  groups = Hash(String, Array(String)).new { |h, k| h[k] = [] of String }
  
  words.each do |word|
    key = word.downcase.chars.sort.join
    groups[key] << word
  end
  
  groups.values.select { |g| g.size > 1 }.sort_by { |g| [-g.size, g.first] }
end

words = ["eat", "tea", "tan", "ate", "nat", "bat", "listen", "silent", "hello"]
groups = group_anagrams(words)
groups.each do |group|
  puts group.inspect
end
# => ["eat", "tea", "ate"]
# => ["tan", "nat"]
# => ["listen", "silent"]
```

---

## 7. Word Frequency

```crystal
# นับความถี่ของคำ
def word_frequency(text : String) : Hash(String, Int32)
  freq = Hash(String, Int32).new(0)
  text.downcase.scan(/\b[a-z]+\b/).each do |m|
    freq[m[0]] += 1
  end
  freq
end

text = "the quick brown fox jumps over the lazy dog the fox"
freq = word_frequency(text)

# เรียงตามความถี่
sorted = freq.to_a.sort_by { |_, count| -count }
sorted.first(5).each do |word, count|
  puts "#{word}: #{count}"
end
# the: 3
# fox: 2
# quick: 1
# brown: 1
# ...

# TF-IDF (Term Frequency - Inverse Document Frequency)
def tf_idf(documents : Array(String)) : Array(Hash(String, Float64))
  # Compute TF for each document
  doc_freqs = documents.map { |doc| word_frequency(doc) }
  
  # Compute IDF
  vocab = doc_freqs.flat_map(&.keys).uniq
  n = documents.size
  
  idf = {} of String => Float64
  vocab.each do |word|
    df = doc_freqs.count { |freq| freq.has_key?(word) }
    idf[word] = Math.log(n.to_f / df.to_f + 1)
  end
  
  # Compute TF-IDF
  doc_freqs.map do |freq|
    total_words = freq.values.sum
    tfidf = {} of String => Float64
    freq.each do |word, count|
      tf = count.to_f / total_words
      tfidf[word] = tf * idf[word]
    end
    tfidf
  end
end

docs = [
  "crystal is a compiled language",
  "crystal is fast and safe",
  "crystal has beautiful syntax",
  "ruby is dynamic and expressive",
]

results = tf_idf(docs)
results.each_with_index do |tfidf, i|
  top = tfidf.to_a.sort_by { |_, v| -v }.first(3)
  puts "Doc #{i + 1} keywords: #{top.map { |w, _| w }.join(", ")}"
end

# N-gram frequency
def ngrams(text : String, n : Int32) : Hash(String, Int32)
  freq = Hash(String, Int32).new(0)
  words = text.downcase.scan(/\b[a-z]+\b/).map(&.[0])
  
  (0..words.size - n).each do |i|
    gram = words[i, n].join(" ")
    freq[gram] += 1
  end
  
  freq
end

text = "to be or not to be that is the question to be"
bigrams = ngrams(text, 2)
bigrams.to_a.sort_by { |_, v| -v }.first(5).each do |gram, count|
  puts "#{gram}: #{count}"
end
# => "to be: 3"
# => ...
```

---

## 8. String Similarity Algorithms

```crystal
# Jaro Similarity
def jaro(s1 : String, s2 : String) : Float64
  return 1.0 if s1 == s2
  return 0.0 if s1.empty? || s2.empty?
  
  max_dist = [s1.size, s2.size].max // 2 - 1
  
  s1_matches = Array.new(s1.size, false)
  s2_matches = Array.new(s2.size, false)
  
  matches = 0
  transpositions = 0
  
  s1.chars.each_with_index do |c1, i|
    start = [0, i - max_dist].max
    finish = [s2.size - 1, i + max_dist].min
    
    (start..finish).each do |j|
      next if s2_matches[j] || s2[j] != c1
      s1_matches[i] = true
      s2_matches[j] = true
      matches += 1
      break
    end
  end
  
  return 0.0 if matches == 0
  
  k = 0
  s1.chars.each_with_index do |c, i|
    next unless s1_matches[i]
    k += 1 while !s2_matches[k]
    transpositions += 1 if c != s2[k]
    k += 1
  end
  
  (matches.to_f / s1.size +
   matches.to_f / s2.size +
   (matches - transpositions / 2).to_f / matches) / 3.0
end

# Jaro-Winkler
def jaro_winkler(s1 : String, s2 : String, p : Float64 = 0.1) : Float64
  j = jaro(s1, s2)
  
  # Count common prefix (up to 4)
  prefix = 0
  [s1.size, s2.size, 4].min.times do |i|
    break unless s1[i] == s2[i]
    prefix += 1
  end
  
  j + prefix * p * (1 - j)
end

puts "%.4f" % jaro("MARTHA", "MARHTA")         # => 0.9444
puts "%.4f" % jaro("DIXON", "DICKSONX")        # => 0.7667
puts "%.4f" % jaro_winkler("MARTHA", "MARHTA") # => 0.9611

# Cosine Similarity สำหรับ text
def cosine_similarity(text1 : String, text2 : String) : Float64
  freq1 = word_frequency(text1)
  freq2 = word_frequency(text2)
  
  all_words = (freq1.keys + freq2.keys).uniq
  
  dot_product = all_words.sum { |w| (freq1[w]? || 0) * (freq2[w]? || 0) }
  
  mag1 = Math.sqrt(freq1.values.sum { |v| v ** 2 })
  mag2 = Math.sqrt(freq2.values.sum { |v| v ** 2 })
  
  return 0.0 if mag1 == 0 || mag2 == 0
  dot_product.to_f / (mag1 * mag2)
end

t1 = "crystal is a compiled language that is fast"
t2 = "crystal is fast and has a nice language design"
t3 = "python is interpreted and dynamic"

puts "%.4f" % cosine_similarity(t1, t2)  # high similarity
puts "%.4f" % cosine_similarity(t1, t3)  # lower similarity
```

---

## 9. Pattern Matching Algorithms

```crystal
# Rabin-Karp Algorithm (rolling hash)
def rabin_karp(text : String, pattern : String) : Array(Int32)
  positions = [] of Int32
  n = text.size
  m = pattern.size
  return positions if m > n
  
  base = 256
  prime = 101
  
  # Calculate hash for pattern and first window
  p_hash = 0
  t_hash = 0
  h = 1
  
  (m - 1).times { h = (h * base) % prime }
  
  m.times do |i|
    p_hash = (base * p_hash + pattern[i].ord) % prime
    t_hash = (base * t_hash + text[i].ord) % prime
  end
  
  (0..n - m).each do |i|
    if p_hash == t_hash
      # Verify (hash collision check)
      if text[i, m] == pattern
        positions << i
      end
    end
    
    if i < n - m
      t_hash = (base * (t_hash - text[i].ord * h) + text[i + m].ord) % prime
      t_hash += prime if t_hash < 0
    end
  end
  
  positions
end

puts rabin_karp("GEEKS FOR GEEKS", "GEEKS").inspect     # => [0, 10]
puts rabin_karp("AABAACAADAABAABA", "AABA").inspect     # => [0, 9, 12]

# Suffix Array (simplified)
def suffix_array(text : String) : Array(Int32)
  n = text.size
  suffixes = (0...n).map { |i| {i, text[i..]} }
  suffixes.sort_by { |_, suf| suf }.map { |i, _| i }
end

sa = suffix_array("banana")
puts sa.inspect  # => [5, 3, 1, 0, 4, 2]  (a, ana, anana, banana, na, nana)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `longest_common_substring` ที่หา substring ยาวที่สุดที่ปรากฏในทั้งสอง strings

### แบบฝึกหัดที่ 2
เขียน `autocomplete` ที่รับ prefix และ dictionary แล้วคืน suggestions โดย rank ตาม frequency

### แบบฝึกหัดที่ 3
เขียน `string_compression` ที่บีบอัด "aaabbbcccc" เป็น "a3b3c4" และ decompress กลับ

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
def longest_common_substring(s1 : String, s2 : String) : String
  m = s1.size
  n = s2.size
  dp = Array.new(m + 1) { Array.new(n + 1, 0) }
  
  max_length = 0
  end_idx = 0
  
  (1..m).each do |i|
    (1..n).each do |j|
      if s1[i - 1] == s2[j - 1]
        dp[i][j] = dp[i - 1][j - 1] + 1
        if dp[i][j] > max_length
          max_length = dp[i][j]
          end_idx = i
        end
      end
    end
  end
  
  s1[end_idx - max_length, max_length]
end

puts longest_common_substring("ABABC", "BABCAB").inspect  # => "BABC"
puts longest_common_substring("crystal", "crystalline").inspect  # => "crystal"

# แบบฝึกหัดที่ 3
def string_compress(str : String) : String
  return str if str.empty?
  
  String.build do |io|
    count = 1
    (1...str.size).each do |i|
      if str[i] == str[i - 1]
        count += 1
      else
        io << str[i - 1]
        io << count if count > 1
        count = 1
      end
    end
    io << str[-1]
    io << count if count > 1
  end
end

def string_decompress(str : String) : String
  String.build do |io|
    i = 0
    while i < str.size
      char = str[i]
      i += 1
      num_start = i
      i += 1 while i < str.size && str[i].ascii_number?
      count = i > num_start ? str[num_start, i - num_start].to_i : 1
      count.times { io << char }
    end
  end
end

compressed = string_compress("aaabbbcccc")
puts compressed  # => "a3b3c4"
puts string_decompress(compressed)  # => "aaabbbcccc"
puts string_compress("aabbcc")  # => "a2b2c2"
puts string_compress("abcde")   # => "abcde" (no compression)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ String Algorithms สำคัญ:

1. **KMP Algorithm** - ค้นหา pattern อย่างมีประสิทธิภาพ O(n + m)
2. **Boyer-Moore-Horspool** - ค้นหาโดย skip characters
3. **Levenshtein Distance** - วัด edit distance ระหว่าง strings
4. **Hamming Distance** - ตำแหน่งที่ต่างกัน
5. **Palindrome Check** - ตรวจสอบและหา palindromes
6. **Anagram Check** - ตรวจสอบและจัดกลุ่ม anagrams
7. **Word Frequency** - นับความถี่และ TF-IDF
8. **Jaro-Winkler** - วัดความคล้ายคลึงของ strings
9. **Rabin-Karp** - hash-based pattern matching

Algorithms เหล่านี้เป็นพื้นฐานของ text processing, search engines, และ bioinformatics
