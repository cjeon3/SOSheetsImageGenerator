# Sound Object Composite Image Generator - Technical Guide

## 📖 Overview

This system generates composite visualizations of sound object drawings by reading data from Google Sheets and creating averaged shape representations. It consists of two main components that work together to transform raw centroid and area data into visual composite images.

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR GOOGLE SHEET                         │
│                                                              │
│  "Data" Tab:                                                │
│  ┌──────────┬─────────┬───────────┬───────┬──────┬────┬────┐│
│  │Participant│Trial #  │Frequency  │Color  │Area  │ X  │ Y  ││
│  ├──────────┼─────────┼───────────┼───────┼──────┼────┼────┤│
│  │Chaebin J │   1     │    31     │ Red   │ 7.23 │-2.5│-0.5││
│  │Chaebin J │   1     │    31     │ Blue  │ 8.55 │-0.1│-1.6││
│  │  ...     │  ...    │   ...     │  ...  │ ...  │... │... ││
│  └──────────┴─────────┴───────────┴───────┴──────┴────┴────┘│
└─────────────────────────────────────────────────────────────┘
                               ↓
                    [Google Apps Script API]
                               ↓
                    Reads 7 columns from Data tab
                    Calculates radius from area
                    Groups by frequency and phase
                               ↓
                    Returns JSON data via Web API
                               ↓
        ┌──────────────────────────────────────────┐
        │  HTML Composite Image Generator          │
        │                                          │
        │  1. Fetches data from API                │
        │  2. Organizes by 11 frequencies          │
        │  3. Separates Red (in-phase) vs Blue     │
        │  4. Draws individual circles             │
        │  5. Calculates & draws averaged circles  │
        │  6. Generates 11 composite images        │
        └──────────────────────────────────────────┘
                               ↓
        ┌──────────────────────────────────────────┐
        │      11 Composite Images (PNG)           │
        │                                          │
        │  31 Hz, 62.5 Hz, 125 Hz, 250 Hz,        │
        │  500 Hz, 1000 Hz, 2000 Hz, 4000 Hz,     │
        │  8000 Hz, 12000 Hz, 16000 Hz            │
        └──────────────────────────────────────────┘
```

---

## 🔧 Component 1: Google Apps Script API

### Purpose
The Google Apps Script acts as a **read-only API** that extracts data from your Google Sheet and serves it in a structured format (JSON) that the HTML viewer can use.

### What It Does

#### 1. Connects to Your Sheet
```javascript
// Opens your specific Google Sheet using the Sheet ID
const ss = SpreadsheetApp.openById(CONFIG.SHEET_ID);

// Accesses the "Data" tab
const sheet = ss.getSheetByName('Data');
```

**How It Works:**
- You provide a unique Sheet ID (found in your Google Sheet's URL)
- The script uses Google's API to open that specific spreadsheet
- It then looks for a tab named "Data" within that spreadsheet

#### 2. Reads Only What's Needed
```javascript
// Column mapping (0-based indexing)
COL_PARTICIPANT: 1,   // Column B
COL_TRIAL: 2,         // Column C  
COL_FREQUENCY: 3,     // Column D
COL_COLOR: 5,         // Column F
COL_AREA: 6,          // Column G
COL_X: 7,             // Column H
COL_Y: 8              // Column I
```

**Why These Columns:**
- **Participant**: Identifies who drew the shape
- **Trial #**: Which trial (1, 2, or 3) this shape is from
- **Frequency**: The sound frequency being visualized (31-16000 Hz)
- **Color**: "Red" (in-phase) or "Blue" (out-of-phase)
- **Area**: The area of the drawn shape in grid squares
- **X, Y**: The centroid (center point) coordinates in ±10 unit space

**What It Skips:**
- Date & Time (Column A) - Not needed for visualization
- dB SPL (Column E) - Not used in composite generation
- Other columns - Not relevant to shape visualization

#### 3. Processes Each Row

For each row in the Data tab, the script:

**Step A: Extracts Data**
```javascript
const participant = String(row[1]).trim();  // "Chaebin J"
const trial = parseFloat(row[2]);           // 1
const frequency = parseFloat(row[3]);       // 31
const color = String(row[5]).trim();        // "Red"
const area = parseFloat(row[6]);            // 7.2254
const centroidX = parseFloat(row[7]);       // -2.4609
const centroidY = parseFloat(row[8]);       // -0.5269
```

**Step B: Validates Data**
```javascript
// Checks that all numeric values are actually numbers
if (isNaN(trial) || isNaN(frequency) || isNaN(area) || 
    isNaN(centroidX) || isNaN(centroidY)) {
  // Skip this row - data is invalid
  return;
}
```

**Why Validation Matters:**
- Empty cells would cause errors
- Text in numeric columns would break calculations
- Ensures data quality

**Step C: Calculates Radius**
```javascript
// Formula: Area = π × r²
// Therefore: r = √(Area / π)
const radius = Math.sqrt(area / Math.PI);
```

**Why We Need Radius:**
- The Data tab stores **area** (how big the shape is)
- To draw a circle, we need **radius** (distance from center to edge)
- We assume shapes are roughly circular for visualization
- This is an approximation that works well for averaged shapes

**Example Calculation:**
```
Given:  Area = 7.2254 grid squares
Step 1: Divide by π: 7.2254 / 3.14159 = 2.2999
Step 2: Take square root: √2.2999 = 1.517
Result: Radius ≈ 1.52 units
```

**Step D: Determines Phase**
```javascript
const phase = color.toLowerCase() === 'red' ? 'red' : 'blue';
```

**What This Means:**
- **Red** = In-phase (sound waves align)
- **Blue** = Out-of-phase (sound waves are inverted)
- This comes from the original drawing tool's color coding

#### 4. Returns Structured Data

The script packages all this information into JSON:

```json
{
  "success": true,
  "count": 925,
  "shapes": [
    {
      "participant": "Chaebin J",
      "trial": 1,
      "frequency": 31,
      "color": "Red",
      "phase": "red",
      "area": 7.2254,
      "centroid": {
        "x": -2.4609,
        "y": -0.5269
      },
      "radius": 1.517
    },
    // ... more shapes
  ],
  "timestamp": "2024-11-20T12:34:56.789Z"
}
```

### API Endpoints

The script provides several endpoints (different ways to query the data):

#### GET ?action=getAllData (Default)
```
Returns: All shapes from all participants, all frequencies
Use: When you want everything (this is what the composite generator uses)
```

#### GET ?action=getSummary
```
Returns: Statistical summary grouped by frequency and phase
Use: Quick overview of data counts and averages
```

#### GET ?action=getParticipant&participant=NAME
```
Returns: All shapes from one specific participant
Use: Filtering data for individual analysis
```

#### GET ?action=getFrequency&frequency=31
```
Returns: All shapes at one specific frequency
Use: Analyzing a single frequency
```

### Security & Privacy

**Read-Only Design:**
- The API functions **never call** any write methods
- `getDataRange().getValues()` only reads, never modifies
- No `.setValues()`, `.appendRow()`, or `.deleteRow()` in API code

**Access Control:**
- Deployed as "Anyone" can access the API
- But they can only READ, never write
- Your Sheet's edit permissions remain unchanged

**What This Means:**
- The HTML viewer can fetch your data
- No one can modify your data through the API
- You control who gets the API URL

---

## 🎨 Component 2: HTML Composite Image Generator

### Purpose
The HTML file is a **web-based application** that fetches data from the Google Apps Script API and generates visual composite images showing averaged shapes at each frequency.

### Coordinate System

Before we dive into visualization, it's important to understand the coordinate system:

```
Y-axis (vertical)
      ↑
  10  │
      │
   5  │
      │
   0  ├─────────────────→ X-axis (horizontal)
      │         5    10
  -5  │
      │
 -10  │

Grid Space: ±10 units in X and Y
Canvas Space: 600×600 pixels
Scale Factor: 30 pixels per unit
```

**Key Conversions:**
```javascript
// To convert from unit coordinates to canvas pixels:
canvasX = center + (unitX × scale)
canvasY = center - (unitY × scale)  // Note: Y is flipped!

// Where:
center = 300 pixels (middle of 600px canvas)
scale = 30 pixels/unit (600px / 20 units)
```

**Why Y is Flipped:**
- In math: positive Y goes up
- On screen: positive Y goes down
- We flip it so math coordinates work naturally

### How It Generates Composites

#### Step 1: Fetch Data from API

```javascript
const response = await fetch(sheetsApiUrl + '?action=getAllData');
const data = await response.json();
// data.shapes contains all 925+ shape records
```

**What Happens:**
1. Browser sends HTTP GET request to your Google Apps Script
2. Script reads Data tab and returns JSON
3. JavaScript receives the data as an object
4. Now we have all shapes in memory

#### Step 2: Organize by Frequency and Phase

```javascript
const participantData = {};

// Create structure for 11 frequencies
FREQUENCIES.forEach(freq => {
  participantData[freq] = { 
    red: [],   // In-phase shapes
    blue: []   // Out-of-phase shapes
  };
});

// Sort each shape into the right frequency and phase
data.shapes.forEach(shape => {
  if (participantData[shape.frequency]) {
    participantData[shape.frequency][shape.phase].push(shape);
  }
});
```

**Result Structure:**
```javascript
{
  31: {
    red: [shape1, shape2, ...],    // All red 31Hz shapes
    blue: [shape3, shape4, ...]    // All blue 31Hz shapes
  },
  62.5: {
    red: [...],
    blue: [...]
  },
  // ... for all 11 frequencies
}
```

**Why This Organization:**
- We need to generate 11 separate images (one per frequency)
- Within each frequency, red and blue are different shapes
- This structure makes it easy to access the right data

#### Step 3: Generate Each Composite Image

For **each of the 11 frequencies**, we:

##### 3A: Create Canvas

```javascript
const canvas = document.createElement('canvas');
canvas.width = 600;
canvas.height = 600;
const ctx = canvas.getContext('2d');
```

**What This Does:**
- Creates a blank 600×600 pixel drawing surface
- Gets a "context" object that lets us draw on it
- Think of it as a blank piece of paper

##### 3B: Draw Background Elements

```javascript
// White background
ctx.fillStyle = 'white';
ctx.fillRect(0, 0, 600, 600);

// Gray circle at 3-unit radius (reference circle)
ctx.beginPath();
ctx.arc(300, 300, 90, 0, 2 * Math.PI);  // 90px = 3 units × 30 px/unit
ctx.strokeStyle = '#cccccc';
ctx.stroke();

// Gray crosshair axes
ctx.strokeStyle = '#dddddd';
ctx.moveTo(0, 300);
ctx.lineTo(600, 300);  // Horizontal line
ctx.moveTo(300, 0);
ctx.lineTo(300, 600);  // Vertical line
ctx.stroke();
```

**Why These Elements:**
- Reference circle: Shows a standard 3-unit radius for scale
- Axes: Help orient the viewer to the coordinate system

##### 3C: Draw Individual Participant Circles

```javascript
function drawIndividualCircles(dataArray, color) {
  ctx.globalAlpha = 0.15;  // Make semi-transparent
  
  dataArray.forEach(shape => {
    // Convert unit coordinates to canvas pixels
    const x = 300 + (shape.centroid.x * 30);
    const y = 300 - (shape.centroid.y * 30);
    const r = shape.radius * 30;
    
    // Draw circle
    ctx.beginPath();
    ctx.arc(x, y, r, 0, 2 * Math.PI);
    ctx.fillStyle = color;
    ctx.fill();
    ctx.strokeStyle = color;
    ctx.stroke();
  });
  
  ctx.globalAlpha = 1.0;  // Reset transparency
}
```

**Example Calculation:**
```
Given shape:
  centroid: { x: -2.4609, y: -0.5269 }
  radius: 1.517

Convert to canvas:
  x = 300 + (-2.4609 × 30) = 300 - 73.827 = 226.173 pixels
  y = 300 - (-0.5269 × 30) = 300 + 15.807 = 315.807 pixels
  r = 1.517 × 30 = 45.51 pixels

Draw circle at (226, 316) with radius 46 pixels
```

**Why Semi-Transparent:**
- With many participants, circles would overlap
- Transparency lets you see all circles
- Darker areas = more overlap/agreement

##### 3D: Calculate Average Circle

This is where the **composite** part happens:

```javascript
function drawAveragedCircle(dataArray, color, isDashed) {
  if (dataArray.length === 0) return;
  
  // Calculate average X coordinate
  const avgX = dataArray.reduce((sum, shape) => {
    return sum + shape.centroid.x;
  }, 0) / dataArray.length;
  
  // Calculate average Y coordinate
  const avgY = dataArray.reduce((sum, shape) => {
    return sum + shape.centroid.y;
  }, 0) / dataArray.length;
  
  // Calculate average radius
  const avgR = dataArray.reduce((sum, shape) => {
    return sum + shape.radius;
  }, 0) / dataArray.length;
  
  // Convert to canvas coordinates
  const x = 300 + (avgX * 30);
  const y = 300 - (avgY * 30);
  const r = avgR * 30;
  
  // Draw averaged circle
  ctx.fillStyle = color + '26';  // Very transparent fill
  ctx.beginPath();
  ctx.arc(x, y, r, 0, 2 * Math.PI);
  ctx.fill();
  
  // Draw bold outline
  ctx.strokeStyle = color;
  ctx.lineWidth = 3;
  if (isDashed) {
    ctx.setLineDash([10, 6]);  // Dashed for blue
  }
  ctx.stroke();
  ctx.setLineDash([]);
  
  // Draw centroid marker
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(x, y, 4, 0, 2 * Math.PI);
  ctx.fill();
}
```

**Averaging Explained:**

Say we have 12 red shapes at 31 Hz:

```
Shape 1: centroid (-2.46, -0.53), radius 1.52
Shape 2: centroid (-1.85, -0.62), radius 1.43
Shape 3: centroid (-2.12, -0.41), radius 1.67
...
Shape 12: centroid (-1.99, -0.55), radius 1.58

Average:
  avgX = (-2.46 + -1.85 + -2.12 + ... + -1.99) / 12 = -2.08
  avgY = (-0.53 + -0.62 + -0.41 + ... + -0.55) / 12 = -0.52
  avgR = (1.52 + 1.43 + 1.67 + ... + 1.58) / 12 = 1.55

Result: Draw circle at (-2.08, -0.52) with radius 1.55
```

**What This Represents:**
- The **average position** where participants drew shapes
- The **average size** of those shapes
- Shows the "typical" or "consensus" shape

##### 3E: Visual Differentiation

**Red vs Blue:**
```javascript
// Red (in-phase): Solid line
ctx.setLineDash([]);
ctx.strokeStyle = '#ff0000';

// Blue (out-of-phase): Dashed line
ctx.setLineDash([10, 6]);  // 10px line, 6px gap
ctx.strokeStyle = '#0000ff';
```

**Why Different Styles:**
- Makes it easy to distinguish phases at a glance
- Red = solid = primary/in-phase
- Blue = dashed = secondary/out-of-phase

##### 3F: Add Title

```javascript
ctx.fillStyle = '#000000';
ctx.font = 'bold 24px Arial';
ctx.textAlign = 'center';
ctx.fillText('31 Hz', 300, 40);

ctx.font = '12px Arial';
ctx.fillStyle = '#666666';
ctx.fillText('Composite from 24 shapes', 300, 60);
```

**Result:**
A complete 600×600 pixel image showing:
- Background reference elements
- All individual participant circles (faint)
- Averaged red circle (solid, bold)
- Averaged blue circle (dashed, bold)
- Title showing frequency

#### Step 4: Display & Download

```javascript
// Add canvas to webpage
const container = document.createElement('div');
container.appendChild(canvas);
document.getElementById('visualization-grid').appendChild(container);

// Make clickable to download
container.addEventListener('click', () => {
  const link = document.createElement('a');
  link.download = `sound_object_composite_31Hz.png`;
  link.href = canvas.toDataURL();  // Convert canvas to PNG
  link.click();
});
```

**What This Does:**
1. Shows the canvas image on the webpage
2. When clicked, converts canvas to PNG file
3. Triggers browser download
4. User gets a high-quality PNG file

#### Step 5: Repeat for All Frequencies

The entire process (steps 3A-3F) repeats 11 times:
- Once for 31 Hz
- Once for 62.5 Hz
- Once for 125 Hz
- ... through 16000 Hz

**Result:** 11 separate composite images

---

## 📊 Statistical Calculations

In addition to images, the system calculates statistics:

### Mean (Average)

```javascript
function mean(arr) {
  return arr.reduce((sum, val) => sum + val, 0) / arr.length;
}
```

**Example:**
```
Areas: [7.23, 8.55, 3.72, 15.36, 9.38]
Sum: 44.24
Count: 5
Mean: 44.24 / 5 = 8.848 grid squares
```

### Standard Deviation (Spread)

```javascript
function stdDev(arr) {
  const avg = mean(arr);
  const squareDiffs = arr.map(val => Math.pow(val - avg, 2));
  return Math.sqrt(mean(squareDiffs));
}
```

**What It Measures:**
- How much values vary from the average
- Low SD = consistent (all similar)
- High SD = variable (wide range)

**Example:**
```
Areas: [7.23, 8.55, 3.72, 15.36, 9.38]
Mean: 8.848

Step 1: Calculate differences from mean
  7.23 - 8.848 = -1.618
  8.55 - 8.848 = -0.298
  3.72 - 8.848 = -5.128
  15.36 - 8.848 = 6.512
  9.38 - 8.848 = 0.532

Step 2: Square each difference
  2.618, 0.089, 26.296, 42.406, 0.283

Step 3: Average the squared differences
  (2.618 + 0.089 + 26.296 + 42.406 + 0.283) / 5 = 14.338

Step 4: Take square root
  √14.338 = 3.787 grid squares
```

**Interpretation:**
```
Mean area: 8.85 ± 3.79 grid squares
This tells us:
- Typical size: 8.85 grid squares
- Variation: shapes range roughly from 5 to 13 grid squares
```

### Statistics Table

For each frequency and phase, the system calculates:

| Statistic | Formula | What It Tells Us |
|-----------|---------|------------------|
| N | count(shapes) | Sample size |
| Mean Area | sum(areas) / N | Average shape size |
| SD Area | stdDev(areas) | Variation in size |
| Mean Radius | sum(radii) / N | Average distance from center |
| SD Radius | stdDev(radii) | Variation in radius |
| Mean X | sum(x_coords) / N | Average horizontal position |
| Mean Y | sum(y_coords) / N | Average vertical position |

---

## 🔍 Understanding the Output

### Reading a Composite Image

```
┌─────────────────────────────────────┐
│             31 Hz                    │
│    Composite from 24 shapes         │
│                                      │
│         ┌─────────┐                 │
│         │ ░░░░░░  │   ← Light circles are
│    ────┼─────●────┼────   individual participants
│         │ ░░░ ●░  │                 │
│         └─────────┘                 │
│                                      │
│  ● = Centroid dot                   │
│  Bold outline = Average shape       │
│  Light fill = Individual shapes     │
└─────────────────────────────────────┘

Legend:
🔴 Red solid line = In-phase
🔵 Blue dashed line = Out-of-phase
```

### What Different Patterns Mean

**Tight Cluster:**
```
     ●●●
    ●●●●●
     ●●●
```
- High agreement between participants
- Consistent shape placement
- Low standard deviation

**Spread Out:**
```
  ●        ●
       ●
  ●        ●
       ●
```
- Variable responses
- Less consensus
- High standard deviation

**Two Groups:**
```
  ●●●      ●●●
  ●●●      ●●●
```
- Bimodal distribution
- Possible subgroups
- May need further analysis

---

## 🧮 Mathematical Foundations

### Circle Approximation

**Why Circles?**
- Original drawings were freeform (any shape)
- We only have centroid and area (not full path)
- Circle is simplest shape defined by center and area
- Works well for averaging many irregular shapes

**Accuracy:**
- Individual circles: Approximate (real shapes may be oval, irregular)
- Averaged circles: More accurate (averaging smooths irregularities)

**Formula Chain:**
```
User draws shape → Tool calculates area → Stored in Data tab
                                            ↓
                              Script calculates radius = √(Area/π)
                                            ↓
                              HTML draws circle with that radius
```

### Coordinate System Math

**Grid Space to Canvas:**
```
Grid: (-10, -10) to (10, 10) = 20 × 20 units
Canvas: (0, 0) to (600, 600) = 600 × 600 pixels

Scale factor = 600 / 20 = 30 pixels per unit

Transform:
  pixel_x = 300 + (unit_x × 30)
  pixel_y = 300 - (unit_y × 30)  // Flip Y axis
```

**Example Points:**
```
Grid → Canvas
(0, 0)     → (300, 300)  Center
(10, 10)   → (600, 0)    Top-right
(-10, -10) → (0, 600)    Bottom-left
(5, 0)     → (450, 300)  Right on X-axis
```

### Averaging Algorithm

**Simple Mean:**
```
Given N shapes with centroids (x₁, y₁), (x₂, y₂), ..., (xₙ, yₙ)

Average centroid:
  x̄ = (x₁ + x₂ + ... + xₙ) / N
  ȳ = (y₁ + y₂ + ... + yₙ) / N
  
Average radius:
  r̄ = (r₁ + r₂ + ... + rₙ) / N
```

**Properties:**
- Minimizes squared distance to all points
- Represents "center of mass" of all shapes
- More participants = more stable average

**Limitations:**
- Outliers pull the average
- Doesn't show distribution shape
- That's why we also calculate standard deviation

---

## 🔐 Security & Data Safety

### What Can Go Wrong?

**Common Concerns:**

❌ **"Can someone hack my sheet through the API?"**
- No. The API is read-only
- No write methods are exposed
- Your Sheet permissions are separate

❌ **"Can someone delete my data?"**
- No. The script never calls delete functions
- Even if they tried, Google Apps Script security prevents it

❌ **"What if I share the API URL accidentally?"**
- Worst case: someone can read your data
- They still can't modify it
- You can revoke the deployment anytime

❌ **"Will this interfere with my data collection?"**
- No. It's completely separate
- Reads don't affect writes
- No race conditions or conflicts

### What The Script CAN'T Do

Even with full permissions, the API functions literally cannot:
- Modify cell values
- Delete rows
- Add rows to Data tab
- Change formulas
- Alter formatting
- Move data around

**Why:**
The code simply doesn't contain those commands. You can verify this yourself by searching the script for write operations.

---

## 🎓 Educational Use Cases

### Research Applications

**Experimental Psychology:**
- Visualize how people perceive different sounds
- Compare in-phase vs out-of-phase perception
- Study frequency-dependent responses

**Data Analysis:**
- Quick composite generation
- Statistical summary tables
- Downloadable publication-ready figures

**Teaching:**
- Show students their collective data
- Demonstrate averaging concepts
- Visualize consensus and variation

### Customization Ideas

**Add More Frequencies:**
```javascript
const FREQUENCIES = [
  31, 62.5, 125, 250, 500, 1000, 2000, 4000, 8000, 12000, 16000,
  20000, 25000  // Add more!
];
```

**Change Colors:**
```javascript
// In drawVisualization function
drawShapes(blueData, '#00ff00', true);  // Green instead of blue
drawShapes(redData, '#ff00ff', false);  // Magenta instead of red
```

**Adjust Canvas Size:**
```javascript
const CANVAS_SIZE = 800;  // Larger images
const UNIT_RANGE = 15;    // Wider range
```

**Export to CSV:**
```javascript
// Add this function to export statistics
function exportToCSV() {
  const csv = participantData.map(row => row.join(',')).join('\n');
  const blob = new Blob([csv], { type: 'text/csv' });
  // Download CSV...
}
```

---

## 📚 Glossary

**API (Application Programming Interface)**
- A way for programs to talk to each other
- Here: Google Sheets ↔ HTML Viewer

**Canvas**
- HTML element for drawing graphics
- Like a digital drawing board

**Centroid**
- The center point (or center of mass) of a shape
- For irregular shapes: weighted average of all points

**Composite**
- Combination of multiple items
- Here: Multiple participants' shapes → one averaged image

**JSON (JavaScript Object Notation)**
- Format for sending data between programs
- Human-readable, machine-parseable

**Phase**
- Relationship between sound waves
- In-phase: waves align (Red)
- Out-of-phase: waves inverted (Blue)

**Radius**
- Distance from center to edge of circle
- Calculated from area: r = √(A/π)

**Read-Only**
- Can look but not change
- API reads data, never writes

**Standard Deviation (SD)**
- Measure of spread/variation
- How much values differ from average

**Unit Coordinates**
- Grid system: ±10 units
- Independent of canvas pixels

---

## 🆘 Troubleshooting

### "No data found"

**Check:**
1. Sheet ID correct?
2. Tab named "Data" exists?
3. Columns in correct order?
4. Data actually in the sheet?

### "Invalid numeric data"

**Cause:**
- Empty cells in numeric columns
- Text where numbers should be
- Special characters

**Fix:**
- Fill all required columns
- Check for #N/A or #ERROR
- Ensure numbers are formatted as numbers

### "Images look wrong"

**Possible Issues:**
1. **All circles tiny:** Areas might be in wrong units
2. **All circles huge:** Scale factor needs adjustment
3. **Circles in wrong positions:** X/Y columns swapped?
4. **No blue shapes:** Check color spelling ("Blue" not "blue")

### "Script timeout"

**Cause:**
- Too much data (>10,000 rows)
- Slow internet connection

**Fix:**
- Process in batches
- Increase timeout limit in Apps Script
- Filter to recent data only

---

## 📝 Summary

### What You've Learned

1. **System Architecture:**
   - Google Sheet (Data source)
   - Apps Script (API middleware)
   - HTML Viewer (Visualization)

2. **Data Flow:**
   - Read 7 columns from Data tab
   - Calculate radius from area
   - Group by frequency and phase
   - Average individual shapes
   - Draw composite images

3. **Key Calculations:**
   - Radius: r = √(A/π)
   - Average position: mean of centroids
   - Average size: mean of radii
   - Variation: standard deviation

4. **Visual Output:**
   - 11 images (one per frequency)
   - Individual circles (semi-transparent)
   - Averaged circles (bold outline)
   - Red solid, blue dashed

### Key Takeaways

✅ **System is read-only and safe**
✅ **Calculations are mathematically sound**
✅ **Process is transparent and verifiable**
✅ **Output is publication-ready**
✅ **Everything is customizable**

---

## 🎉 You're Ready!

You now understand:
- How data flows through the system
- How calculations are performed
- What the composite images represent
- How to interpret the results
- How to troubleshoot issues

**Next Steps:**
1. Deploy the Google Apps Script
2. Open the HTML Composite Generator
3. Generate your 11 composite images
4. Analyze the results!

---

*For setup instructions, see DEPLOYMENT_CHECKLIST.md*
*For updates, see UPDATES_SUMMARY.md*
*For security details, see DATA_SAFETY.md*
