<template>
  <div class="app-container">
    <h1>Dice Roller with History</h1>

    <!-- Input and Roll button (unchanged) -->
    <div class="input-section">
      <label for="numDiceInput">Number of dice (6-sided):</label>
      <input id="numDiceInput" type="number" min="1" v-model="numDice" />
      <button @click="rollDice">Roll</button>
    </div>

    <!-- Displayed results / chart -->
    <div v-if="displayedResults.length" class="results-section">
      <h3>Distribution:</h3>

      <div class="chart-container">
        <div
          class="bar-wrapper"
          v-for="(freq, index) in displayedDistribution"
          :key="index"
        >
          <!-- Animated bar -->
          <div
            class="bar"
            :ref="(el) => (bars[index] = el)"
            :data-value="freq"
          ></div>

          <!-- Dice icons with Font Awesome -->
          <div class="bar-label">
            <i :class="diceFaIcons[index]" class="dice-fa-icon"></i>
            ({{ freq }})
          </div>
        </div>
      </div>

      <!-- SUMMARY as big buttons: 2+, 3+, 4+, 5+ -->
      <div class="summary-section">
        <!-- If sum2Plus > 0, show a big button for rolling sum2Plus dice -->
        <button
          v-if="sum2Plus > 0"
          class="big-btn"
          @click="rollDiceWithCount(sum2Plus)"
        >
          2+ ({{ sum2Plus }})
        </button>

        <!-- 3+ -->
        <button
          v-if="sum3Plus > 0"
          class="big-btn"
          @click="rollDiceWithCount(sum3Plus)"
        >
          3+ ({{ sum3Plus }})
        </button>

        <!-- 4+ -->
        <button
          v-if="sum4Plus > 0"
          class="big-btn"
          @click="rollDiceWithCount(sum4Plus)"
        >
          4+ ({{ sum4Plus }})
        </button>

        <!-- 5+ -->
        <button
          v-if="sum5Plus > 0"
          class="big-btn"
          @click="rollDiceWithCount(sum5Plus)"
        >
          5+ ({{ sum5Plus }})
        </button>
      </div>

      <!-- Reroll buttons (existing logic) -->
      <div class="reroll-section" v-if="currentRoll && !currentRoll.hasReroll">
        <button v-if="displayedDistribution[0] > 0" @click="reRollOnes">
          ReRoll 1s
        </button>
        <button @click="reRollX">ReRoll X</button>
      </div>
    </div>

    <!-- History list (unchanged) -->
    <div v-if="rollHistory.length" class="history-section">
      <h2>History</h2>
      <ul>
        <li
          v-for="(roll, index) in rollHistory"
          :key="roll.id"
          :class="{ 'active-history': index === selectedRollIndex }"
          @click="selectRoll(index)"
        >
          <i
            v-if="roll.hasReroll"
            class="fa-solid fa-redo white"
            title="Rerolled"
          ></i>
          Roll {{ rollHistory.length - index }}
          ({{ roll.results.length }} dice)
          <span class="timestamp">
            ({{ roll.timestamp }})
          </span>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import { ref, reactive, watch, computed, nextTick } from "vue";
import anime from "animejs/lib/anime.es.js";

function rollSingleDie() {
  const array = new Uint32Array(1);
  crypto.getRandomValues(array);
  return (array[0] % 6) + 1;
}

export default {
  setup() {
    // Basic reactive state (same as before)
    const numDice = ref(1);
    const rollHistory = ref([]);
    const selectedRollIndex = ref(null);
    const displayedResults = ref([]);
    const displayedDistribution = reactive([0, 0, 0, 0, 0, 0]);
    const bars = [];

    const diceFaIcons = [
      "fa-solid fa-dice-one",
      "fa-solid fa-dice-two",
      "fa-solid fa-dice-three",
      "fa-solid fa-dice-four",
      "fa-solid fa-dice-five",
      "fa-solid fa-dice-six",
    ];

    const currentRoll = computed(() => {
      if (selectedRollIndex.value === null) return null;
      return rollHistory.value[selectedRollIndex.value] || null;
    });

    // 1) Standard rollDice method
    async function rollDice() {
      const results = [];
      const distribution = [0, 0, 0, 0, 0, 0];

      for (let i = 0; i < numDice.value; i++) {
        const value = rollSingleDie();
        results.push(value);
        distribution[value - 1] += 1;
      }

      const now = new Date();
      const timestamp = now.toLocaleTimeString();

      rollHistory.value.unshift({
        id: Date.now() + Math.random(),
        timestamp,
        results,
        distribution,
        hasReroll: false,
      });

      selectedRollIndex.value = null;
      await nextTick();
      selectedRollIndex.value = 0;
    }

    // 2) NEW: rollDiceWithCount(count) for creating a new roll with 'count' dice
    async function rollDiceWithCount(count) {
      // If user tries to roll 0 dice, do nothing
      if (count <= 0) return;

      const results = [];
      const distribution = [0, 0, 0, 0, 0, 0];

      for (let i = 0; i < count; i++) {
        const value = rollSingleDie();
        results.push(value);
        distribution[value - 1] += 1;
      }

      const now = new Date();
      const timestamp = now.toLocaleTimeString();

      // Insert as a brand-new roll in the history
      rollHistory.value.unshift({
        id: Date.now() + Math.random(),
        timestamp,
        results,
        distribution,
        hasReroll: false,
      });

      selectedRollIndex.value = null;
      await nextTick();
      selectedRollIndex.value = 0;
    }

    function selectRoll(index) {
      selectedRollIndex.value = index;
    }

    function animateBars() {
      const containerEl = document.querySelector(".chart-container");
      if (!containerEl) return;

      const containerHeightPx = parseFloat(
        window.getComputedStyle(containerEl).height
      );

      displayedDistribution.forEach((freq, i) => {
        const rawHeight = freq * 12;
        const finalHeight = Math.min(rawHeight, containerHeightPx);

        anime({
          targets: bars[i],
          height: finalHeight + "px",
          duration: 800,
          easing: "easeOutCubic",
        });
      });
    }

    async function reRollOnes() {
      if (!currentRoll.value || currentRoll.value.hasReroll) return;

      const numOnes = displayedDistribution[0];
      if (numOnes <= 0) return;

      displayedDistribution[0] = 0;
      const keptResults = displayedResults.value.filter((val) => val !== 1);

      // Reroll
      const newRolls = [];
      for (let i = 0; i < numOnes; i++) {
        newRolls.push(rollSingleDie());
      }

      displayedResults.value = [...keptResults, ...newRolls];
      for (const val of newRolls) {
        displayedDistribution[val - 1] += 1;
      }

      currentRoll.value.results = displayedResults.value.slice();
      for (let i = 0; i < 6; i++) {
        currentRoll.value.distribution[i] = displayedDistribution[i];
      }
      currentRoll.value.hasReroll = true;

      await nextTick();
      animateBars();
    }

    async function reRollX() {
      if (!currentRoll.value || currentRoll.value.hasReroll) return;

      const inputValue = prompt("Enter a face value (1 to 6) to re-roll:");
      if (!inputValue) return;

      const x = parseInt(inputValue, 10);
      if (isNaN(x) || x < 1 || x > 6) {
        alert("Invalid input. Must be 1..6.");
        return;
      }

      let rerollCount = 0;
      for (let face = 1; face <= x; face++) {
        rerollCount += displayedDistribution[face - 1];
        displayedDistribution[face - 1] = 0;
      }

      const keptResults = displayedResults.value.filter((val) => val > x);

      const newRolls = [];
      for (let i = 0; i < rerollCount; i++) {
        newRolls.push(rollSingleDie());
      }
      displayedResults.value = [...keptResults, ...newRolls];
      for (const val of newRolls) {
        displayedDistribution[val - 1] += 1;
      }

      currentRoll.value.results = displayedResults.value.slice();
      for (let i = 0; i < 6; i++) {
        currentRoll.value.distribution[i] = displayedDistribution[i];
      }
      currentRoll.value.hasReroll = true;

      await nextTick();
      animateBars();
    }

    // Watcher for selectedRoll => load distribution
    watch(selectedRollIndex, async (newIndex) => {
      if (newIndex === null) return;

      const roll = rollHistory.value[newIndex];
      if (!roll) return;

      displayedResults.value = roll.results;
      for (let i = 0; i < 6; i++) {
        displayedDistribution[i] = roll.distribution[i];
      }

      await nextTick();
      animateBars();
    });

    // Existing computed summaries
    const sum2Plus = computed(() => {
      return (
        displayedDistribution[1] +
        displayedDistribution[2] +
        displayedDistribution[3] +
        displayedDistribution[4] +
        displayedDistribution[5]
      );
    });
    const sum3Plus = computed(() => {
      return (
        displayedDistribution[2] +
        displayedDistribution[3] +
        displayedDistribution[4] +
        displayedDistribution[5]
      );
    });
    const sum4Plus = computed(() => {
      return (
        displayedDistribution[3] +
        displayedDistribution[4] +
        displayedDistribution[5]
      );
    });
    const sum5Plus = computed(() => {
      return displayedDistribution[4] + displayedDistribution[5];
    });

    return {
      numDice,
      rollHistory,
      selectedRollIndex,
      displayedResults,
      displayedDistribution,
      bars,
      diceFaIcons,

      currentRoll,
      rollDice,
      rollDiceWithCount, // <--- NEW method for rolling a custom count
      selectRoll,
      reRollOnes,
      reRollX,

      // Summaries for 2+..5+
      sum2Plus,
      sum3Plus,
      sum4Plus,
      sum5Plus,
    };
  },
};
</script>

<style scoped>
.app-container {
  margin: 0 auto;
  max-width: 600px;
  padding: 20px;
  font-family: sans-serif;
  text-align: center;
}

.input-section {
  margin-bottom: 20px;
}

.results-section {
  border-top: 1px solid #ddd;
  padding-top: 20px;
}

/* Chart container: 30vh tall, with animated bars */
.chart-container {
  display: flex;
  justify-content: center;
  align-items: flex-end;
  margin-top: 30px;
  gap: 10px;
  height: 30vh;
}

.bar-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-end;
}

.bar {
  width: 30px;
  background-color: #3eaf7c;
  margin-bottom: 5px;
  height: 0px; /* start from zero */
  transition: height 0.2s ease-out;
}

.bar-label {
  font-weight: bold;
  display: flex;
  align-items: center;
  gap: 6px; /* space between icon & freq */
}

.dice-fa-icon {
  font-size: 24px;
}

/* Big button styling for the summary buttons */
.summary-section {
  margin-top: 20px;
  display: flex;
  gap: 10px;
  justify-content: center;
  flex-wrap: wrap; /* allow wrapping if needed */
}

.big-btn {
  padding: 12px 20px;
  font-size: 1rem;
  background-color: #28a745;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  min-width: 80px;
  transition: background-color 0.2s;
}

.big-btn:hover {
  background-color: #218838;
}

.reroll-section {
  margin-top: 20px;
  display: flex;
  gap: 10px;
  justify-content: center;
}

.history-section {
  border-top: 1px solid #ddd;
  padding-top: 20px;
  margin-bottom: 20px;
}

.history-section ul {
  list-style: none;
  padding: 0;
}

.history-section li {
  cursor: pointer;
  padding: 5px 10px;
  margin: 5px auto;
  max-width: 400px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.history-section li:hover {
  background-color: #7da2ff56;
}

.active-history {
  background-color: #7da2ff56;
  border-color: #7da2ff;
}

.timestamp {
  font-size: 0.9rem;
  color: #ccc;
  margin-left: 5px;
}
</style>
