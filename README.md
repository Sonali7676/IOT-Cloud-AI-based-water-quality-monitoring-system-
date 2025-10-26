<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Water Quality Monitoring System</title>
<style>
  body {
    font-family: Arial, sans-serif;
    margin: 1rem;
    background-color: #eef7fb;
    color: #333;
    text-align: center;
  }
  h1 {
    color: #006699;
  }
  #image-container {
    margin: 1rem auto;
    max-width: 500px;
  }
  #uploaded-image {
    max-width: 100%;
    border-radius: 8px;
    border: 2px solid #0077cc;
  }
  #quality-result {
    margin-top: 1rem;
    font-weight: bold;
    font-size: 1.2rem;
    color: #004466;
  }
  #upload-label {
    display: inline-block;
    margin-top: 1rem;
    padding: 0.5rem 1rem;
    background-color: #0077cc;
    color: white;
    border-radius: 4px;
    cursor: pointer;
  }
  #upload-input {
    display: none;
  }
  table {
    margin: 1rem auto;
    border-collapse: collapse;
  }
  th, td {
    border: 1px solid #0077cc;
    padding: 0.5rem 1rem;
  }
  th {
    background-color: #0077cc;
    color: white;
  }
  footer {
    margin-top: 3rem;
    font-size: 0.9rem;
    color: #777;
  }
</style>
</head>
<body>
<h1>Water Quality Monitoring System</h1>
<p>Upload a water image or use the default image below to analyze water quality.</p>

<label id="upload-label" for="upload-input">Upload Water Image</label>
<input type="file" id="upload-input" accept="image/*" />

<div id="image-container">
  <img id="uploaded-image" src="https://images.pexels.com/photos/132037/pexels-photo-132037.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940" alt="Water" />
</div>

<div id="quality-result">Analyzing water quality...</div>

<table id="analysis-table" class="hidden">
  <thead>
    <tr>
      <th>Attribute</th>
      <th>Result</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Average Color (RGB)</td>
      <td id="avg-color"></td>
    </tr>
    <tr>
      <td>Clarity Estimate</td>
      <td id="clarity"></td>
    </tr>
    <tr>
      <td>Particles Estimate</td>
      <td id="particles"></td>
    </tr>
    <tr>
      <td>Drinking Water Suitable</td>
      <td id="drinkable"></td>
    </tr>
  </tbody>
</table>

<script>
  const img = document.getElementById('uploaded-image');
  const qualityResult = document.getElementById('quality-result');
  const uploadInput = document.getElementById('upload-input');

  const analysisTable = document.getElementById('analysis-table');
  const avgColorTd = document.getElementById('avg-color');
  const clarityTd = document.getElementById('clarity');
  const particlesTd = document.getElementById('particles');
  const drinkableTd = document.getElementById('drinkable');

  function analyzeWaterQuality(image) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    const width = image.naturalWidth;
    const height = image.naturalHeight;
    canvas.width = width;
    canvas.height = height;
    ctx.drawImage(image, 0, 0, width, height);
    try {
      const imageData = ctx.getImageData(0, 0, width, height);
      const data = imageData.data;

      let totalPixels = width * height;
      let sumR = 0;
      let sumG = 0;
      let sumB = 0;
      let brightnessSum = 0;
      let darkPixelCount = 0;
      let particleCount = 0;

      // Thresholds
      const darkPixelThreshold = 50; // brightness below this is dark (possible particles)
      const particlePixelThreshold = 60; // Red channel significance for particles

      for (let i = 0; i < data.length; i += 4) {
        const r = data[i];
        const g = data[i + 1];
        const b = data[i + 2];
        const brightness = 0.299 * r + 0.587 * g + 0.114 * b;

        sumR += r;
        sumG += g;
        sumB += b;
        brightnessSum += brightness;

        if (brightness < darkPixelThreshold) {
          darkPixelCount++;
        }

        // Simple particle detection heuristic:
        // If pixel is dark with low blue but more red/yellow, might indicate particles or impurities
        // We'll count pixels where brightness is low and red channel is relatively high
        if (brightness < particlePixelThreshold && r > g && r > b) {
          particleCount++;
        }
      }

      const avgR = Math.round(sumR / totalPixels);
      const avgG = Math.round(sumG / totalPixels);
      const avgB = Math.round(sumB / totalPixels);
      const avgBrightness = brightnessSum / totalPixels;

      // Clarity estimate based on amount of dark pixels (higher dark pixels -> less clear)
      const darkPixelRatio = darkPixelCount / totalPixels;
      let clarity;
      if (darkPixelRatio < 0.02 && avgBrightness > 100) {
        clarity = 'Clear';
      } else if (darkPixelRatio < 0.1 && avgBrightness > 70) {
        clarity = 'Moderate';
      } else {
        clarity = 'Low';
      }

      // Particles estimate based on particleCount ratio
      const particleRatio = particleCount / totalPixels;
      let particles;
      if (particleRatio < 0.005) {
        particles = 'Very Low';
      } else if (particleRatio < 0.02) {
        particles = 'Moderate';
      } else {
        particles = 'High';
      }

      // Decide drinkable or not (very simplified logic):
      // Drinkable only if clarity is Clear or Moderate and particles Very Low or Moderate
      // Not drinkable if clarity Low or particles High
      let drinkable;
      if ((clarity === 'Clear' || clarity === 'Moderate') && (particles === 'Very Low' || particles === 'Moderate')) {
        drinkable = 'Yes';
      } else {
        drinkable = 'No';
      }

      return {
        avgColor: `rgb(${avgR}, ${avgG}, ${avgB})`,
        clarity,
        particles,
        drinkable
      };
    } catch (e) {
      // Security error (CORS) or other issues with image data access
      return null;
    }
  }

  function updateQuality() {
    if (!img.complete) {
      qualityResult.textContent = 'Loading image...';
      img.onload = () => {
        const result = analyzeWaterQuality(img);
        displayResult(result);
      };
    } else {
      const result = analyzeWaterQuality(img);
      displayResult(result);
    }
  }

  function displayResult(result) {
    if (!result) {
      qualityResult.textContent = 'Cannot analyze this image due to browser security restrictions.';
      analysisTable.classList.add('hidden');
      return;
    }
    qualityResult.textContent = '';
    analysisTable.classList.remove('hidden');

    avgColorTd.textContent = result.avgColor;
    avgColorTd.style.backgroundColor = result.avgColor;
    avgColorTd.style.color = getContrastYIQ(result.avgColor);

    clarityTd.textContent = result.clarity;
    particlesTd.textContent = result.particles;
    drinkableTd.textContent = result.drinkable;

    // Color drinkable cell green or red
    drinkableTd.style.color = result.drinkable === 'Yes' ? 'green' : 'red';
  }

  function getContrastYIQ(rgbString) {
    // Parse rgb string like "rgb(12, 34, 56)"
    const rgb = rgbString.match(/\d+/g);
    if (!rgb) return '#000';
    const r = parseInt(rgb[0]);
    const g = parseInt(rgb[1]);
    const b = parseInt(rgb[2]);
    const yiq = ((r*299)+(g*587)+(b*114))/1000;
    return (yiq >= 128) ? 'black' : 'white';
  }

  window.onload = updateQuality;

  uploadInput.addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = function(evt) {
        img.src = evt.target.result;
      };
      reader.readAsDataURL(file);
      qualityResult.textContent = 'Analyzing uploaded image...';
      img.onload = () => {
        updateQuality();
      };
    }
  });
</script>

<footer>
  <p>Note: Analysis is a simple visual approximation and does not replace professional water quality testing.</p>
</footer>
</body>
</html>
