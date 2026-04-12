# Rocket Simulator Refactoring Summary

## Overview
The rocket simulator has been refactored to support advanced multi-stage rocket simulations with altitude-dependent aerodynamics and time-based thrust curves.

---

## 1. Multi-Stage Rocket Support

### New `RocketStage` Class
```javascript
class RocketStage {
  constructor(dryMass, propMass, burnTime, isp, cd, diameter, thrustCurveType='linear')
}
```

**Features:**
- Encapsulates all stage-specific parameters
- Calculates cross-sectional area automatically
- Implements time-based thrust curves via `getThrust(timeInBurn, totalBurnTime)`

### Sequential Stage Simulation
- **Stage 1** (always active): Initial rocket stage
- **Stage 2** (optional): Separates when Stage 1 fuel is exhausted
- **Stage 3** (optional): Separates when Stage 2 fuel is exhausted
- Each stage has independent:
  - Dry mass and propellant mass
  - Burn time and specific impulse (Isp)
  - Drag coefficient (Cd)
  - Separation mass (shed at separation)

### Dynamic Mass Updates
- Total mass starts with sum of all stages
- After each stage burnout:
  - Stage separation mass is shed
  - Previous stage dry mass is jettisoned
  - Remaining stages continue acceleration
- Formula: `mass = Σ(remaining stage masses) - separation mass`

---

## 2. Altitude-Dependent Drag Calculation

### Air Density Function
```javascript
function airDensity(alt) {
  const altMeters = Math.max(0, alt);
  return RHO0 * Math.exp(-altMeters / H_ATM);
}
```

**Constants:**
- `RHO0 = 1.225 kg/m³` (sea level density)
- `H_ATM = 8500 m` (scale height of atmosphere)

**Physics:**
- Uses exponential atmosphere model: `ρ(h) = ρ₀ × e^(-h/H)`
- Density decreases ~12% per 1000m altitude
- Recalculated at **every timestep** (dt = 0.01s)

### Drag Force Equation
```javascript
const drag = 0.5 * rho * Cd * A * v²
```

Applied component-wise:
```javascript
fx -= drag * (vx / spd);  // Horizontal drag
fy -= drag * (vy / spd);  // Vertical drag
```

**Impact on Flight:**
- Reducing density at altitude reduces drag significantly
- Enables realistic high-altitude performance
- At 20km: density ≈ 5% of sea level (negligible drag)
- At 5km: density ≈ 55% of sea level

### Wind Force (Also Altitude-Dependent)
```javascript
const windF = 0.5 * rho * 0.5 * A * (wind - vx) * |wind - vx|
```

---

## 3. Time-Based Thrust Curves

### Thrust Curve Types

#### 1. Constant Thrust
- Thrust remains constant throughout burn
- Classic model motor behavior
- Formula: `F = Isp × g₀ × (m_prop / t_burn)`

#### 2. Linear Ramp (Bell Curve)
- Thrust ramps linearly to peak at midpoint
- Then linearly decays to zero at burnout
- More realistic for many real motors
```javascript
// First half: ramp up
F(t) = F_peak × (t / t_burn) / 0.5

// Second half: ramp down
F(t) = F_peak × (1 - t / t_burn) / 0.5
```

#### 3. Bell (Gaussian)
- Gaussian-like curve with smooth rise and decay
- Realistic for advanced motor designs
- Formula: `F(t) = F_peak × exp(-(x² / 2))` where `x = (t/t_burn - 0.5) × 4`

### Implementation
```javascript
function getThrust(stage, timeInBurn) {
  if (timeInBurn < 0 || timeInBurn > stage.burnTime) return 0;
  return stage.getThrust(timeInBurn, stage.burnTime);
}
```

**Usage in Simulation:**
- Called every timestep during burn phase
- Interpolates thrust based on elapsed burn time
- Applied both vertically and at launch angle

---

## 4. User Interface Updates

### New "Multi-Stage & Thrust Curves" Panel

#### Thrust Curve Selection
```
Thrust Curve Type: [Constant | Linear Ramp | Bell]
```

#### Stage 2 Parameters
- Propellant Mass (kg)
- Dry Mass (kg)
- Specific Impulse (seconds)
- Burn Time (auto-calculated if 0)
- Separation Mass (kg shed at separation)

#### Stage 3 Parameters
- Same as Stage 2
- Third stage separates after Stage 2 burnout

### Default Values
- Stage 2 Isp: 70 seconds (typical small engine)
- Stage 2 Dry Mass: 0.03 kg
- Stage 3 Dry Mass: 0.02 kg
- Separation masses configurable per stage

---

## 5. Simulation Engine Changes

### Main Loop (pseudocode)
```javascript
while (t < MAX_TIME) {
  // 1. Recalculate atmosphere at current altitude
  rho = airDensity(y);
  g = gravity(y);
  
  // 2. Check for stage separation
  if (propLeft <= 0 && currentStage < maxStages) {
    Separate current stage;
    Activate next stage;
  }
  
  // 3. Get thrust from current stage's thrust curve
  thrust = getThrust(currentStage, timeInBurn);
  Apply forces (thrust - drag - gravity - wind);
  
  // 4. Integration (Euler method)
  v += a * dt;
  pos += v * dt;
  t += dt;
}
```

### Key Improvements
- **Altitude-aware**: All forces vary with altitude
- **Multi-stage**: Seamless stage transitions
- **Dynamic thrust**: Thrust curves enable realistic motor simulation
- **Accurate mass**: Total mass decreases as fuel burns and stages separate

---

## 6. Delta-V Calculation

Updated to sum all stages:
```javascript
const totalDeltaV = stages.reduce((sum, s) => 
  sum + s.isp * G0 * Math.log((s.dryMass + s.propMass) / s.dryMass), 0
);
```

Uses Tsiolkovsky equation for each stage, then sums them.

---

## Example Scenarios

### Single-Stage Rocket
- Configure only Stage 1
- Set Stage 2 Propellant Mass = 0
- Flies like original simulator

### Two-Stage Rocket
- Stage 1: High thrust, short burn (booster)
- Stage 2: Lower thrust, longer burn (sustainer)
- Automatic separation at Stage 1 burnout
- Total Isp higher than single-stage equivalent

### High-Altitude Flight
- Thrust curves matter: choose "Bell" for smoother acceleration
- Drag reduces by 98% at 40km (minimal air density)
- Wind force similarly reduced at altitude

---

## Validation

All changes maintain backward compatibility:
- Single-stage mode works identically to original
- Antimatter mode unchanged
- Chart rendering and telemetry preserved
- CSV export includes all flight data

## Physics Notes

- Simulation uses DT = 0.01s timesteps
- Air density model valid up to ~100km
- Gravity corrected for altitude
- All forces integrated using first-order Euler method
