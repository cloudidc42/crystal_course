# Part 28 - หน่วยวัด (Units of Measure)

## บทนำ (Introduction)

Units of Measure เป็น F# feature พิเศษที่ทำให้ compiler ตรวจสอบว่าเราไม่ได้ผสมหน่วยวัดที่ไม่ถูกต้อง เช่น บวกเมตรกับวินาที เป็น compile-time safety ที่มีประสิทธิภาพมาก

---

## 28.1 [<Measure>] Attribute

```fsharp
// ============ นิยาม Units of Measure ============
[<Measure>] type m      // meter
[<Measure>] type s      // second
[<Measure>] type kg     // kilogram
[<Measure>] type A      // ampere
[<Measure>] type K      // kelvin
[<Measure>] type mol    // mole
[<Measure>] type cd     // candela

// ============ ใช้งาน ============
let height = 1.8<m>
let time = 60.0<s>
let mass = 75.0<kg>

printfn "Height: %A" height   // 1.8
printfn "Time: %A" time       // 60.0
printfn "Mass: %A" mass       // 75.0

// ============ Type Annotations ============
let speed : float<m/s> = 10.0<m/s>
let force : float<kg m/s^2> = 9.8<kg m/s^2>

// ============ Integer units ============
let count : int<m> = 5<m>
printfn "Count: %d" count

// ============ Incompatible types (COMPILE ERRORS) ============
// let wrong = height + time  // ERROR: m + s
// let wrong2 = height * time  // OK! = m*s
// let wrong3 : float<m> = 1.0<s>  // ERROR: type mismatch
```

---

## 28.2 Arithmetic with Units

```fsharp
// ============ การคำนวณด้วยหน่วย ============
[<Measure>] type m
[<Measure>] type s
[<Measure>] type kg

// Addition / Subtraction: units ต้องเหมือนกัน
let d1 = 100.0<m>
let d2 = 50.0<m>
let total = d1 + d2     // 150.0<m>
let diff = d1 - d2      // 50.0<m>

printfn "total: %A m" total
printfn "diff: %A m" diff

// Multiplication: units คูณกัน
let t = 10.0<s>
let area = d1 * d2      // 5000.0<m^2>
let distance = 5.0<m/s> * t  // 50.0<m>  (m/s * s = m)

printfn "area: %A m^2" area
printfn "distance: %A m" distance

// Division: units หารกัน
let speed = d1 / t     // 10.0<m/s>
printfn "speed: %A m/s" speed

// ============ Complex calculations ============
let velocity = 30.0<m/s>
let mass = 5.0<kg>
let kineticEnergy = 0.5 * mass * velocity * velocity
// 0.5 * kg * m/s * m/s = kg*m^2/s^2 = Joule

printfn "KE: %A kg*m^2/s^2" kineticEnergy

// Power
let time2 = 3.0<s>
let power = kineticEnergy / time2
// kg*m^2/s^2 / s = kg*m^2/s^3 = Watt

printfn "Power: %A W" power

// ============ Square root ============
open System

let area2 = 16.0<m^2>
let side = sqrt area2   // 4.0<m>  -- sqrt preserves units correctly
printfn "Side: %A m" side

// ============ Comparing values with units ============
let d3 = 200.0<m>
let d4 = 100.0<m>
printfn "d3 > d4: %b" (d3 > d4)  // true
// Cannot compare different units

// ============ Functions with units ============
let circleArea (r: float<m>) : float<m^2> =
    Math.PI * r * r

let circumference (r: float<m>) : float<m> =
    2.0 * Math.PI * r

printfn "Area of r=5m: %.2f m^2" (circleArea 5.0<m>)
printfn "Circumference of r=5m: %.2f m" (circumference 5.0<m>)
```

---

## 28.3 Unit Conversion

```fsharp
// ============ Unit Conversion ============
[<Measure>] type m
[<Measure>] type cm
[<Measure>] type km
[<Measure>] type ft
[<Measure>] type inch_
[<Measure>] type s
[<Measure>] type min_
[<Measure>] type h
[<Measure>] type kg
[<Measure>] type g
[<Measure>] type lb

// Conversion factors
let mToCm (x: float<m>) : float<cm> = x * 100.0<cm/m>
let cmToM (x: float<cm>) : float<m> = x / 100.0<cm/m>
let mToKm (x: float<m>) : float<km> = x / 1000.0<m/km>
let kmToM (x: float<km>) : float<m> = x * 1000.0<m/km>
let mToFt (x: float<m>) : float<ft> = x * 3.28084<ft/m>
let ftToM (x: float<ft>) : float<m> = x / 3.28084<ft/m>
let ftToInch (x: float<ft>) : float<inch_> = x * 12.0<inch_/ft>

let sToMin (x: float<s>) : float<min_> = x / 60.0<s/min_>
let minToH (x: float<min_>) : float<h> = x / 60.0<min_/h>
let sToH (x: float<s>) : float<h> = x / 3600.0<s/h>

let kgToG (x: float<kg>) : float<g> = x * 1000.0<g/kg>
let kgToLb (x: float<kg>) : float<lb> = x * 2.20462<lb/kg>

// ============ ใช้งาน ============
let height = 1.8<m>
printfn "%.2f m = %.1f cm = %.4f km = %.2f ft" 
    (float height)
    (float (mToCm height))
    (float (mToKm height))
    (float (mToFt height))

let time = 7200.0<s>
printfn "%.0f s = %.0f min = %.2f h" 
    (float time)
    (float (sToMin time))
    (float (sToH time))

let weight = 75.0<kg>
printfn "%.1f kg = %.0f g = %.2f lb" 
    (float weight)
    (float (kgToG weight))
    (float (kgToLb weight))

// ============ Type-safe conversion pipeline ============
let marathonInFt =
    42.195<km>
    |> (fun km -> km * 1000.0<m/km>)  // km -> m
    |> mToFt

printfn "Marathon (42.195 km) = %.0f ft" (float marathonInFt)

// ============ Temperature conversion ============
[<Measure>] type degC
[<Measure>] type degF
[<Measure>] type degK

let celsiusToFahrenheit (c: float<degC>) : float<degF> =
    (float c * 9.0 / 5.0 + 32.0) * 1.0<degF>

let fahrenheitToCelsius (f: float<degF>) : float<degC> =
    ((float f - 32.0) * 5.0 / 9.0) * 1.0<degC>

let celsiusToKelvin (c: float<degC>) : float<degK> =
    (float c + 273.15) * 1.0<degK>

printfn "100°C = %.1f°F = %.2f K" 
    (float (celsiusToFahrenheit 100.0<degC>))
    (float (celsiusToKelvin 100.0<degC>))

printfn "98.6°F = %.1f°C" (float (fahrenheitToCelsius 98.6<degF>))
```

---

## 28.4 Derived Units

```fsharp
// ============ Derived Units ============
[<Measure>] type m
[<Measure>] type s
[<Measure>] type kg

// SI Derived Units
[<Measure>] type N = kg m / s^2       // Newton (force)
[<Measure>] type J = kg m^2 / s^2     // Joule (energy)
[<Measure>] type W = kg m^2 / s^3     // Watt (power)
[<Measure>] type Pa = kg / (m s^2)    // Pascal (pressure)
[<Measure>] type Hz = s^-1            // Hertz (frequency)

// คำนวณโดยใช้ derived units
let force = 10.0<N>
let displacement = 5.0<m>
let work = force * displacement  // N * m = kg m^2 / s^2 = J

printfn "Work: %A J" work

let power_usage = 100.0<W>
let time_use = 3600.0<s>
let energy_consumed = power_usage * time_use  // W * s = J

printfn "Energy: %A J" energy_consumed

// ============ Custom Domain Units ============
[<Measure>] type pixel
[<Measure>] type dot          // printer dot
[<Measure>] type dpi = dot/inch_

// Assuming 96 DPI screen
[<Measure>] type inch_

let screenWidth = 1920.0<pixel>
let screenHeight = 1080.0<pixel>
let pixelDensity = 96.0<dpi>

// Pixels per inch conversion
let pixelsToInches (px: float<pixel>) (dpi: float<dpi>) : float<inch_> =
    px / (dpi * 1.0<pixel/dot>)  // simplified

// ============ Currency units ============
[<Measure>] type THB   // Thai Baht
[<Measure>] type USD   // US Dollar
[<Measure>] type EUR   // Euro

let exchangeRateUsdToThb = 35.5<THB/USD>
let exchangeRateEurToUsd = 1.08<USD/EUR>

let usdAmount = 100.0<USD>
let thbAmount = usdAmount * exchangeRateUsdToThb
let eurAmount = 50.0<EUR>
let thbFromEur = eurAmount * exchangeRateEurToUsd * exchangeRateUsdToThb

printfn "$%.2f = ฿%.2f" (float usdAmount) (float thbAmount)
printfn "€%.2f = ฿%.2f" (float eurAmount) (float thbFromEur)
```

---

## 28.5 Checking Unit Safety

```fsharp
// ============ Unit Safety Examples ============
[<Measure>] type m
[<Measure>] type s
[<Measure>] type kg

// ============ ตัวอย่าง: Physics calculations ============

// Kinematics: v = u + at
let calcFinalVelocity (u: float<m/s>) (a: float<m/s^2>) (t: float<s>) : float<m/s> =
    u + a * t   // m/s + m/s^2 * s = m/s + m/s = m/s ✓

// Position: s = ut + 0.5*a*t^2
let calcPosition (u: float<m/s>) (a: float<m/s^2>) (t: float<s>) : float<m> =
    u * t + 0.5 * a * t * t   // m/s*s + m/s^2*s^2 = m + m = m ✓

// Newton's second law: F = ma
let calcForce (m: float<kg>) (a: float<m/s^2>) : float<kg m/s^2> =
    m * a   // kg * m/s^2 ✓

// Gravitational potential energy: PE = mgh
let calcPE (mass: float<kg>) (g: float<m/s^2>) (h: float<m>) : float<kg m^2/s^2> =
    mass * g * h   // kg * m/s^2 * m = kg*m^2/s^2 ✓

// ============ ใช้งาน ============
let g = 9.8<m/s^2>

let v0 = 0.0<m/s>
let a = 2.5<m/s^2>
let t = 10.0<s>

let vf = calcFinalVelocity v0 a t
printfn "Final velocity: %.1f m/s" (float vf)   // 25.0

let pos = calcPosition v0 a t
printfn "Position: %.1f m" (float pos)           // 125.0

let mass = 5.0<kg>
let force = calcForce mass a
printfn "Force: %.1f N" (float force)           // 12.5

let height = 10.0<m>
let pe = calcPE mass g height
printfn "Potential Energy: %.1f J" (float pe)   // 490.0

// ============ Prevention of bugs ============
// The following would be COMPILE ERRORS:
// let wrong1 = v0 + h      // m/s + m ≠ valid
// let wrong2 = mass * t    // kg * s ≠ m
// let wrong3 : float<m> = pos + vf  // m ≠ m/s

// Without units, these bugs are easy to make:
let distanceWithoutUnits v0_raw a_raw t_raw = v0_raw + 0.5 * a_raw * t_raw  // WRONG formula!
let distanceWithUnits (v0: float<m/s>) (a: float<m/s^2>) (t: float<s>) : float<m> = 
    v0 * t + 0.5 * a * t * t  // Correct - compiler ensures

printfn "Without units (BUG): %f" (distanceWithoutUnits 0.0 2.5 10.0)  // 12.5 (wrong!)
printfn "With units (correct): %f m" (float (distanceWithUnits v0 a t))  // 125.0
```

---

## 28.6 Removing Units

```fsharp
// ============ Removing Units ============
// บางครั้งต้องส่งค่าให้ functions ที่ไม่รู้จัก units

[<Measure>] type m
[<Measure>] type s
[<Measure>] type kg

// ============ float function ============
let height = 1.8<m>
let rawHeight : float = float height   // strip units
printfn "Raw height: %f" rawHeight

// ============ LanguagePrimitives ============
open Microsoft.FSharp.Core

// float to measure
let fromFloat (x: float) : float<m> = x * 1.0<m>
let withUnit (x: float) : float<m> = LanguagePrimitives.FloatWithMeasure x

let myHeight = withUnit 1.8
printfn "Height with unit: %A" myHeight

// ============ Stripping for external libraries ============
let sendToApi (value: float<m>) =
    let rawValue = float value  // strip units for API call
    sprintf "{ \"distance\": %f, \"unit\": \"meters\" }" rawValue

printfn "%s" (sendToApi height)

// ============ Round-trip: strip and reattach ============
let processAndReturn (value: float<m>) =
    let raw = float value
    let processed = raw * 2.0 + 0.5   // some processing without units
    LanguagePrimitives.FloatWithMeasure<m> processed  // reattach units

let processed = processAndReturn height
printfn "Processed: %A m" processed

// ============ Safe conversion function ============
let convertUnit<[<Measure>] 'From, [<Measure>] 'To> 
    (factor: float<'To/'From>) 
    (value: float<'From>) : float<'To> =
    value * factor

[<Measure>] type km
let mToKm = convertUnit<m, km> (1.0<km> / 1000.0<m>)
let distance = 5000.0<m>
printfn "%.1f m = %.1f km" (float distance) (float (mToKm distance))
```

---

## 28.7 Units in Generic Functions

```fsharp
// ============ Generic Functions with Units ============
[<Measure>] type m
[<Measure>] type s
[<Measure>] type kg

// Generic function ที่ทำงานกับ any unit
let square (x: float<'u>) : float<'u^2> = x * x

let doubleValue (x: float<'u>) : float<'u> = x * 2.0

let average (values: float<'u> list) : float<'u> =
    let sum = List.sum values
    sum / float (List.length values)

// ============ ใช้งาน ============
let length = 5.0<m>
let squaredLength = square length
printfn "5m squared: %A m^2" squaredLength

let doubled = doubleValue 3.0<kg>
printfn "Doubled: %A kg" doubled

let lengths = [1.0<m>; 2.0<m>; 3.0<m>; 4.0<m>]
let avgLength = average lengths
printfn "Average: %A m" avgLength

// ============ Type-safe vector ============
type Vector3<[<Measure>] 'u> = {
    X: float<'u>
    Y: float<'u>
    Z: float<'u>
}

let vectorAdd (v1: Vector3<'u>) (v2: Vector3<'u>) : Vector3<'u> = {
    X = v1.X + v2.X
    Y = v1.Y + v2.Y
    Z = v1.Z + v2.Z
}

let vectorScale (scalar: float) (v: Vector3<'u>) : Vector3<'u> = {
    X = v.X * scalar
    Y = v.Y * scalar
    Z = v.Z * scalar
}

let vectorMagnitude (v: Vector3<'u>) : float<'u> =
    sqrt (v.X * v.X + v.Y * v.Y + v.Z * v.Z)

let dotProduct (v1: Vector3<'u>) (v2: Vector3<'u>) : float<'u^2> =
    v1.X * v2.X + v1.Y * v2.Y + v1.Z * v2.Z

// ============ ใช้งาน ============
let pos1 = { X = 1.0<m>; Y = 2.0<m>; Z = 3.0<m> }
let pos2 = { X = 4.0<m>; Y = 5.0<m>; Z = 6.0<m> }
let velocity = { X = 1.0<m/s>; Y = 0.5<m/s>; Z = 0.0<m/s> }

let sum = vectorAdd pos1 pos2
printfn "Sum: %A" sum

let mag = vectorMagnitude pos1
printfn "Magnitude: %.4f m" (float mag)

let scaled = vectorScale 2.0 velocity
printfn "Scaled velocity: %A" scaled

// ============ Constraint: units cannot mix ============
// vectorAdd pos1 velocity  // COMPILE ERROR!  m ≠ m/s
// vectorAdd pos1 { X = 1.0<s>; ... }  // COMPILE ERROR!
```

---

## 28.8 Real-World Physics Calculations

```fsharp
// ============ Physics Module ============
[<Measure>] type m
[<Measure>] type s
[<Measure>] type kg
[<Measure>] type K      // Kelvin
[<Measure>] type mol

// ============ Constants ============
let g = 9.80665<m/s^2>         // gravitational acceleration
let G = 6.674e-11<m^3/(kg s^2)> // gravitational constant  
let c = 299792458.0<m/s>        // speed of light
let k_B = 1.380649e-23<kg m^2/(s^2 K)>  // Boltzmann constant

// ============ Kinematics ============
module Kinematics =
    let finalVelocity (v0: float<m/s>) (a: float<m/s^2>) (t: float<s>) : float<m/s> =
        v0 + a * t
    
    let displacement (v0: float<m/s>) (a: float<m/s^2>) (t: float<s>) : float<m> =
        v0 * t + 0.5 * a * t * t
    
    let finalVelocityFromDistance (v0: float<m/s>) (a: float<m/s^2>) (d: float<m>) : float<m/s> =
        sqrt (v0 * v0 + 2.0 * a * d)
    
    let projectileRange (v0: float<m/s>) (angle: float) : float<m> =
        let v0x = v0 * cos angle
        let v0y = v0 * sin angle
        let t = 2.0 * v0y / g
        v0x * t

// ============ Dynamics ============
module Dynamics =
    let force (mass: float<kg>) (accel: float<m/s^2>) : float<kg m/s^2> =
        mass * accel
    
    let momentum (mass: float<kg>) (velocity: float<m/s>) : float<kg m/s> =
        mass * velocity
    
    let kineticEnergy (mass: float<kg>) (velocity: float<m/s>) : float<kg m^2/s^2> =
        0.5 * mass * velocity * velocity
    
    let potentialEnergy (mass: float<kg>) (height: float<m>) : float<kg m^2/s^2> =
        mass * g * height
    
    let gravitationalForce (m1: float<kg>) (m2: float<kg>) (r: float<m>) : float<kg m/s^2> =
        G * m1 * m2 / (r * r)

// ============ ใช้งาน ============
printfn "=== Projectile Motion ==="
let speed = 20.0<m/s>
let angle = System.Math.PI / 4.0  // 45 degrees
let range = Kinematics.projectileRange speed angle
printfn "Range at 45°: %.2f m" (float range)

printfn "\n=== Free Fall ==="
let height = 100.0<m>
let fallTime = sqrt (2.0 * height / g)
let impactVelocity = g * fallTime
printfn "Fall from %.0f m: time=%.2f s, impact=%.2f m/s" 
    (float height) (float fallTime) (float impactVelocity)

printfn "\n=== Orbital Mechanics ==="
let earthMass = 5.972e24<kg>
let earthRadius = 6.371e6<m>
let orbitalAltitude = 400e3<m>   // ISS altitude
let orbitalRadius = earthRadius + orbitalAltitude
let orbitalSpeed = sqrt (G * earthMass / orbitalRadius)
let orbitalPeriod = 2.0 * System.Math.PI * orbitalRadius / orbitalSpeed
printfn "ISS orbital speed: %.0f m/s" (float orbitalSpeed)
printfn "ISS orbital period: %.1f min" (float orbitalPeriod / 60.0)

printfn "\n=== Energy Conservation ==="
let ball_mass = 0.5<kg>
let throw_height = 5.0<m>
let throw_speed = 3.0<m/s>
let pe = Dynamics.potentialEnergy ball_mass throw_height
let ke = Dynamics.kineticEnergy ball_mass throw_speed
let total_energy = pe + ke
printfn "PE = %.2f J, KE = %.2f J, Total = %.2f J"
    (float pe) (float ke) (float total_energy)
```

---

## 28.9 Financial Calculations with Units

```fsharp
// ============ Financial Units ============
[<Measure>] type USD
[<Measure>] type THB
[<Measure>] type EUR
[<Measure>] type JPY
[<Measure>] type GBP

[<Measure>] type year
[<Measure>] type percent    // percentage point

// ============ Interest Calculations ============
let simpleInterest 
    (principal: float<USD>) 
    (rate: float<percent/year>) 
    (time: float<year>) : float<USD> =
    principal * float rate / 100.0 * float time

let compoundInterest 
    (principal: float<USD>) 
    (rate: float<percent/year>) 
    (time: float<year>) 
    (n: float) : float<USD> =   // n = compounding periods per year
    let r = float rate / 100.0 / n
    let t = float time * n
    principal * (1.0 + r) ** t

let futureValue = compoundInterest 10000.0<USD> 5.0<percent/year> 10.0<year> 12.0

printfn "Future Value (5%% compound, 10yr): $%.2f" (float futureValue)

// ============ Currency Exchange ============
type ExchangeRate<[<Measure>] 'From, [<Measure>] 'To> = float<'To/'From>

let usdToThb : ExchangeRate<USD, THB> = 35.5<THB/USD>
let usdToEur : ExchangeRate<USD, EUR> = 0.92<EUR/USD>
let eurToJpy : ExchangeRate<EUR, JPY> = 160.0<JPY/EUR>

let exchange (rate: ExchangeRate<'From, 'To>) (amount: float<'From>) : float<'To> =
    amount * rate

let salary = 5000.0<USD>
let thbSalary = exchange usdToThb salary
let eurSalary = exchange usdToEur salary

printfn "Salary: $%.2f = ฿%.2f = €%.2f" 
    (float salary) (float thbSalary) (float eurSalary)

// ============ Stock Portfolio ============
[<Measure>] type share

type Stock = {
    Ticker: string
    Price: float<USD/share>
    Shares: float<share>
}

let portfolioValue (stocks: Stock list) : float<USD> =
    stocks |> List.sumBy (fun s -> s.Price * s.Shares)

let portfolio = [
    { Ticker = "AAPL"; Price = 175.0<USD/share>; Shares = 100.0<share> }
    { Ticker = "GOOGL"; Price = 130.0<USD/share>; Shares = 50.0<share> }
    { Ticker = "MSFT"; Price = 380.0<USD/share>; Shares = 75.0<share> }
]

let totalValue = portfolioValue portfolio
printfn "\nPortfolio value: $%.2f" (float totalValue)
printfn "In THB: ฿%.2f" (float (exchange usdToThb totalValue))

// ============ Loan Calculator ============
let monthlyPayment 
    (principal: float<USD>) 
    (annualRate: float<percent/year>) 
    (termMonths: int) : float<USD> =
    let monthlyRate = float annualRate / 100.0 / 12.0
    let n = float termMonths
    principal * (monthlyRate * (1.0 + monthlyRate) ** n) / ((1.0 + monthlyRate) ** n - 1.0)

let loanAmount = 500000.0<USD>  // $500,000 mortgage
let rate = 4.5<percent/year>
let term = 360  // 30 years

let payment = monthlyPayment loanAmount rate term
printfn "\nMortgage: $%.0f at %.1f%% for %d months" 
    (float loanAmount) (float rate) term
printfn "Monthly payment: $%.2f" (float payment)
printfn "Total paid: $%.2f" (float payment * float term)
printfn "Total interest: $%.2f" (float payment * float term - float loanAmount)
```

---

## 28.10 Type-Safe API Design with Units

```fsharp
// ============ Type-Safe API Design ============
[<Measure>] type px      // pixels
[<Measure>] type em      // em units
[<Measure>] type rem     // root em
[<Measure>] type vw      // viewport width %
[<Measure>] type vh      // viewport height %
[<Measure>] type pt      // points
[<Measure>] type deg     // degrees
[<Measure>] type rad     // radians
[<Measure>] type ms      // milliseconds
[<Measure>] type opacity // 0.0 to 1.0

// CSS-like styling API
type Color = { R: byte; G: byte; B: byte; A: float<opacity> }

type FontSize =
    | Px of float<px>
    | Em of float<em>
    | Rem of float<rem>

type StyleProperty =
    | FontSizeStyle of FontSize
    | Width of float<px>
    | Height of float<px>
    | Opacity of float<opacity>
    | TransitionDuration of float<ms>
    | Rotation of float<deg>

// Validation
let validateOpacity (o: float<opacity>) : Result<float<opacity>, string> =
    if float o < 0.0 || float o > 1.0 then Error "Opacity must be 0.0-1.0"
    else Ok o

let validateColor r g b (a: float<opacity>) : Result<Color, string> =
    match validateOpacity a with
    | Error e -> Error e
    | Ok opacity -> Ok { R = r; G = g; B = b; A = opacity }

// ============ Animation with units ============
type Keyframe = {
    Time: float<ms>
    Position: float<px> * float<px>
    Rotation: float<deg>
    Scale: float
}

let interpolate (t: float) (kf1: Keyframe) (kf2: Keyframe) : Keyframe =
    let lerp a b = a + (b - a) * t
    let lerpPx (a: float<px>) (b: float<px>) = a + (b - a) * t
    let lerpDeg (a: float<deg>) (b: float<deg>) = a + (b - a) * t
    
    {
        Time = lerp (float kf1.Time) (float kf2.Time) * 1.0<ms>
        Position = 
            (lerpPx (fst kf1.Position) (fst kf2.Position),
             lerpPx (snd kf1.Position) (snd kf2.Position))
        Rotation = lerpDeg kf1.Rotation kf2.Rotation
        Scale = lerp kf1.Scale kf2.Scale
    }

let kf1 = { Time = 0.0<ms>; Position = (0.0<px>, 0.0<px>); Rotation = 0.0<deg>; Scale = 1.0 }
let kf2 = { Time = 1000.0<ms>; Position = (100.0<px>, 50.0<px>); Rotation = 90.0<deg>; Scale = 2.0 }

let midpoint = interpolate 0.5 kf1 kf2
printfn "Midpoint position: (%.1f, %.1f) px" 
    (float (fst midpoint.Position)) (float (snd midpoint.Position))
printfn "Midpoint rotation: %.1f deg" (float midpoint.Rotation)

// ============ Network API ============
[<Measure>] type bytes
[<Measure>] type kbps    // kilobits per second
[<Measure>] type mb      // megabytes

let bytesToMb (b: float<bytes>) : float<mb> = b / 1048576.0<bytes/mb>
let mbToBytes (m: float<mb>) : float<bytes> = m * 1048576.0<bytes/mb>

let calculateDownloadTime (fileSize: float<mb>) (speed: float<kbps>) : float<s> =
    let fileSizeKb = float fileSize * 8192.0  // mb to kb
    let speedKb = float speed
    (fileSizeKb / speedKb) * 1.0<s>

let file = 250.0<mb>
let connection = 50000.0<kbps>  // 50 Mbps

let downloadTime = calculateDownloadTime file connection
printfn "\nDownload %.0f MB at %.0f kbps = %.1f s" 
    (float file) (float connection) (float downloadTime)
```

---

## สรุป (Summary)

```
Units of Measure ใน F#:

การนิยาม:
[<Measure>] type m     // simple unit
[<Measure>] type N = kg m / s^2  // derived unit

การใช้งาน:
let x = 5.0<m>
let speed = 10.0<m/s>

Arithmetic:
- +, -: ต้องเป็น units เดียวกัน
- *: units คูณกัน
- /: units หารกัน
- sqrt: units square root

Generic functions:
let square (x: float<'u>) : float<'u^2> = x * x

Removing units:
let raw = float value
let withUnit = LanguagePrimitives.FloatWithMeasure<m> raw

Benefits:
- Compile-time safety ไม่มี unit mismatch
- Documentation ใน code
- Prevent Mars Climate Orbiter-type bugs
- ไม่มี overhead ที่ runtime
```

---

*จบ Part 28 - หน่วยวัด (Units of Measure)*
