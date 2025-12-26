<script setup>
import { ref } from 'vue';

// --- STATE ---
const calcMode = ref('calculatePace');

// Shared Inputs
const eventType = ref('custom');
const distance = ref('');
const unit = ref('mi');

// Time Inputs
const hrs = ref('');
const mins = ref(0);
const secs = ref(0);

// Pace Inputs
const paceMins = ref(8);
const paceSecs = ref(0);

// Results & UI
const results = ref(null);
const isCopied = ref(false);

// --- HELPERS ---
const formatTime = (totalSeconds) => {
  let h = Math.floor(totalSeconds / 3600);
  let m = Math.floor((totalSeconds % 3600) / 60);
  let s = Math.round(totalSeconds % 60);

  if (s === 60) { s = 0; m++; }
  if (m === 60) { m = 0; h++; }

  const minStr = m < 10 && h > 0 ? "0" + m : m;
  const secStr = s < 10 ? "0" + s : s;

  if (h > 0) return `${h}:${minStr}:${secStr}`;
  return `${m}:${secStr}`;
};

const handleEventChange = () => {
  const val = eventType.value;
  if (val !== 'custom') {
    distance.value = parseFloat(val);
    unit.value = 'km';
  } else {
    distance.value = '';
    unit.value = 'mi';
  }
};

const handleCopy = async () => {
  if (!results.value) return;

  // 1. Get Distance Text
  const distDisplay = `${distance.value} ${unit.value}`;

  // 2. Get Time Text
  let timeDisplay = "";
  if (calcMode.value === 'calculatePace') {
    // If calculating pace, time is what user entered
    const h = parseInt(hrs.value) || 0;
    const m = parseInt(mins.value) || 0;
    const s = parseInt(secs.value) || 0;
    const totalSecs = (h * 3600) + (m * 60) + s;
    timeDisplay = formatTime(totalSecs);
  } else {
    // If calculating time, time is the result
    timeDisplay = results.value.mainMetric;
  }

  // 3. Build String
  let text = `${distDisplay} : ${timeDisplay}\n`;
  
  results.value.splits.forEach(row => {
    text += `${row.label} - ${row.pace}\n`;
  });

  try {
    await navigator.clipboard.writeText(text);
    isCopied.value = true;
    setTimeout(() => isCopied.value = false, 2000);
  } catch (err) {
    console.error('Failed to copy', err);
  }
};

// --- CORE LOGIC ---
const handleCalculate = () => {
  isCopied.value = false;

  const dInput = parseFloat(distance.value);
  if (!dInput || dInput <= 0) {
    alert("Please enter a valid distance");
    return;
  }

  const totalMiles = unit.value === 'km' ? dInput * 0.621371 : dInput;
  let avgSecondsPerMile = 0;
  let totalDurationSeconds = 0;

  // MODE A: Know Time -> Calc Pace
  if (calcMode.value === 'calculatePace') {
    const h = parseInt(hrs.value) || 0;
    const m = parseInt(mins.value) || 0;
    const s = parseInt(secs.value) || 0;
    totalDurationSeconds = (h * 3600) + (m * 60) + s;

    if (totalDurationSeconds === 0) {
      alert("Please enter a goal time");
      return;
    }
    avgSecondsPerMile = totalDurationSeconds / totalMiles;
  } 
  // MODE B: Know Pace -> Calc Time
  else {
    const pM = parseInt(paceMins.value) || 0;
    const pS = parseInt(paceSecs.value) || 0;
    if (pM === 0 && pS === 0) {
      alert("Please enter a target pace");
      return;
    }
    avgSecondsPerMile = (pM * 60) + pS;
    totalDurationSeconds = avgSecondsPerMile * totalMiles;
  }

  // Generate Splits
  const cutdown = 5;
  const fullMiles = Math.floor(totalMiles);
  const partialMile = totalMiles - fullMiles;
  
  let currentPaceSeconds = avgSecondsPerMile + ((totalMiles * cutdown) / 2);
  let accumulatedSeconds = 0;
  const newSplits = [];

  for (let i = 1; i <= fullMiles; i++) {
    let splitTime = currentPaceSeconds;
    accumulatedSeconds += splitTime;
    newSplits.push({
      label: i.toString(),
      pace: formatTime(splitTime),
      elapsed: formatTime(accumulatedSeconds)
    });
    currentPaceSeconds -= cutdown;
  }

  if (partialMile > 0.01) {
    let partialTime = currentPaceSeconds * partialMile;
    accumulatedSeconds += partialTime;
    newSplits.push({
      label: `${fullMiles + 1} (${partialMile.toFixed(2)})`,
      pace: formatTime(currentPaceSeconds),
      elapsed: formatTime(accumulatedSeconds)
    });
  }

  results.value = {
    mainMetric: calcMode.value === 'calculatePace' 
      ? formatTime(avgSecondsPerMile) 
      : formatTime(totalDurationSeconds),
    mainLabel: calcMode.value === 'calculatePace' 
      ? "Average Pace Required" 
      : "Total Estimated Time",
    subLabel: calcMode.value === 'calculatePace' 
      ? "per mile" 
      : `at ${formatTime(avgSecondsPerMile)}/mi avg`,
    splits: newSplits
  };
};
</script>

<template>
  <div class="calculator-card">
    <h2>Pace Calculator</h2>

    <div class="mode-toggle">
      <button 
        :class="{ active: calcMode === 'calculatePace' }" 
        @click="calcMode = 'calculatePace'"
      >
        I Know My Time
      </button>
      <button 
        :class="{ active: calcMode === 'calculateTime' }" 
        @click="calcMode = 'calculateTime'"
      >
        I Know My Pace
      </button>
    </div>

    <div class="form-group">
      <label>Event / Distance</label>
      <select v-model="eventType" @change="handleEventChange">
        <option value="custom">Custom Distance</option>
        <option value="5">5K</option>
        <option value="10">10K</option>
        <option value="21.0975">Half Marathon</option>
        <option value="42.195">Marathon</option>
      </select>
      
      <div class="unit-toggle">
        <input 
          type="number" 
          inputmode="decimal" 
          v-model="distance" 
          placeholder="Distance" 
          step="0.01"
        >
        <select v-model="unit" style="width: 80px;">
          <option value="mi">mi</option>
          <option value="km">km</option>
        </select>
      </div>
    </div>

    <div v-if="calcMode === 'calculatePace'" class="form-group">
      <label>Goal Time (Hr : Min : Sec)</label>
      <div class="time-inputs">
        <div class="time-field">
          <input type="number" inputmode="numeric" v-model="hrs" placeholder="00">
        </div>
        <div class="time-separator">:</div>
        <div class="time-field">
          <input type="number" inputmode="numeric" v-model="mins" placeholder="00">
        </div>
        <div class="time-separator">:</div>
        <div class="time-field">
          <input type="number" inputmode="numeric" v-model="secs" placeholder="00">
        </div>
      </div>
    </div>

    <div v-if="calcMode === 'calculateTime'" class="form-group">
      <label>Desired Average Pace (min/mi)</label>
      <div class="time-inputs">
        <div class="time-field">
          <input type="number" inputmode="numeric" v-model="paceMins" placeholder="8">
        </div>
        <div class="time-separator">:</div>
        <div class="time-field">
          <input type="number" inputmode="numeric" v-model="paceSecs" placeholder="00">
        </div>
      </div>
    </div>

    <button class="calculate-btn" @click="handleCalculate">
      {{ calcMode === 'calculatePace' ? 'Calculate Pace' : 'Calculate Time' }}
    </button>

    <div v-if="results" class="results">
      <div style="position: relative;">
        <button 
          @click="handleCopy"
          :style="{
            position: 'absolute',
            right: 0,
            top: 0,
            padding: '5px 10px',
            fontSize: '0.8rem',
            borderRadius: '6px',
            border: '1px solid #ddd',
            background: isCopied ? '#2ecc71' : 'white',
            color: isCopied ? 'white' : '#333',
            cursor: 'pointer',
            transition: 'all 0.2s',
            zIndex: 10
          }"
        >
          {{ isCopied ? 'Copied!' : 'Copy Splits' }}
        </button>

        <div class="result-summary">
          <span class="sub-label">{{ results.mainLabel }}</span>
          <span class="main-pace">{{ results.mainMetric }}</span>
          <span class="sub-label">{{ results.subLabel }}</span>
        </div>
      </div>

      <h3>Negative Split Strategy</h3>
      <p style="text-align: center; font-size: 0.85rem; color: #666; margin-bottom: 15px;">
        Strategy: Start ~2.5s slower, finish ~2.5s faster than avg
      </p>

      <table>
        <thead>
          <tr>
            <th>Mile</th>
            <th>Split Pace</th>
            <th>Elapsed Time</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, index) in results.splits" :key="index">
            <td>{{ row.label }}</td>
            <td class="negative-trend">{{ row.pace }}</td>
            <td>{{ row.elapsed }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>