<template>
  <div class="cryptoquip-container">
    <div class="title">Secret Agent</div>
    <div class="second-title">{{ new Date().toLocaleDateString() }}</div>
    <div class="index">{{ entry }}</div>
    <div class="letter-container">
      <div class="hello">Hello,</div>
      <div class="instructions">Todays mission: decode this phrase. This is a Caesar Cipher, also known as a Shift Cipher. In this puzzle, each letter has been replaced by another, and the entire alphabet has been shifted by a certain number of positions. For example, if the alphabet is shifted 3 places to the right, A becomes D, B becomes E, and so on. The key to solving this puzzle is figuring out how many spaces the alphabet has been shifted. The relationship between letters is consistent throughout the puzzle. For instance, if X equals O, that pattern will apply to every instance of X in the puzzle. You can make your guess for each box by typing directly under the existing letter (see: the question marks). If you get stuck, click “Unlock Hint” for more help. Good Luck. </div>
    <!-- <div class="hint">YOUR HINT IS: {{ currentHint }}</div> -->

    <div class="puzzle">
      <div v-for="(char, index) in codedPhrase" :key="index" class="puzzle-char">
        <span class="cipher">{{ char }}</span>

        <input
          v-if="char.match(/[A-Z]/) && !isPuzzleLocked && !isGiveUp"
          type="text"
          maxlength="1"
          v-model="guesses[index]"
          @input="checkWin"
          class="input-box"
          placeholder="?"
        />
        <span v-else class="non-letter">{{ char }}</span>
      </div>
    </div>

    <div class="progress">
      {{ decodedText }}
    </div>
    <div class="hint-column">
      <button class="hint-btn" v-if="additionalHints.length > 0" @click="showHint" :disabled="isSolved || isPuzzleLocked || isGiveUp">
        Unlock Hint
      </button>
      <div v-for="(hint, index) in additionalHints" :key="index" class="hint-box" :class="{'locked': index >= hintIndex}">
        Hint {{ index + 1 }}: {{ index < hintIndex ? hint : 'locked' }}
      </div>
    </div>

    <div class="give-up" v-if="hintIndex === additionalHints.length && !isPuzzleLocked && !isSolved && !isGiveUp">
      <button class="give-up-btn" @click="giveUp" :disabled="isPuzzleLocked">
        Give Up?
      </button>
    </div>
  </div>

  <div class="streak">
    <div>Current Streak: {{ currentStreak }}</div>
    <div>Highest Streak: {{ highestStreak }}</div>
  </div>

  <div v-if="isPuzzleLocked || newPuzzleReady" class="modal-overlay">
    <div class="modal">
      <div v-if="isSolved">
        <h2>Great Work.</h2>
        <p>Phrase: {{ originalText }}</p>
        <p>Mission Complete Streak: {{ currentStreak }}</p>
        <p>Your next mission is in: {{ countdown }}</p>
      </div>
      <div v-else-if="isGiveUp">
        <h2>Try again tomorrow!</h2>
        <p>Phrase: {{ originalText }}</p>
        <p>Streak: {{ currentStreak }}</p>
        <p>Next puzzle in: {{ countdown }}</p>
      </div>

      <div v-else-if="newPuzzleReady" class="new-puzzle-button">
    <h2>New Puzzle Available</h2>
    <button class="btn-new-puzzle" @click="resetPuzzle">Start New Puzzle</button>
  </div>

      <button @click="generateShareResult" v-if="isSolved || isGiveUp" class="share-button">
        Share Result
      </button>
    </div>
  </div>
</div>
</template>
  
  <script>
export default {
  data() {
    return {
      puzzles: [],
      currentPuzzle: null,
      originalText: "",
      codedPhrase: "",
      currentHint: "",
      additionalHints: [],
      currentHintToShow: "",
      guesses: [],
      isSolved: false,
      isGiveUp: false,
      hintIndex: 0,
      countdown: null, 
      currentStreak: 0,
      highestStreak: 0,
      giveUpMessage: "",
      isPuzzleLocked: false,
      modalContent: "",
      entry: "",
      newPuzzleReady: false,
      notificationMessage: '',
    };
  },
  computed: {
    decodedText() {
      return this.codedPhrase
        .split("")
        .map((char, index) =>
          char.match(/[A-Z]/) ? (this.guesses[index] || "_").toUpperCase() : char
        )
        .join("");
    },
  },
  methods: {
    async loadPuzzles() {
  try {
    const response = await fetch("/puzzles.json");
    this.puzzles = await response.json();
    this.selectPuzzleForToday();
    this.loadStreaks();
  } catch (error) {
    console.error("Failed to load puzzles:", error);
  }
},
selectPuzzleForToday() {
  const currentDate = new Date().toISOString().split("T")[0];
  console.log("Current Date: ", currentDate);

  const puzzleLocked = localStorage.getItem("puzzleLocked") === "true";
  const isGiveUp = localStorage.getItem("isGiveUp") === "true";
  const isSolved = localStorage.getItem("isSolved") === "true"; 

  this.isSolved = isSolved;
  this.isGiveUp = isGiveUp;

  if (puzzleLocked) {
    this.isPuzzleLocked = true;
    this.startCountdownToMidnight();
  } else if (isGiveUp) {
    this.giveUpMessage = "You gave up. Try again next time!";
    this.startCountdownToMidnight();
  } else if (isSolved) {
    this.startCountdownToMidnight();
  } else {


    const puzzle = this.puzzles.find(p => p.date === currentDate);

    if (puzzle) {
      this.currentPuzzle = puzzle;
      this.originalText = puzzle.phrase;
      this.currentHint = puzzle.hint;
      this.codedPhrase = puzzle.coded_phrase;
      this.additionalHints = puzzle.additional_hints || [];
      this.hintIndex = 0;
      this.entry = puzzle.entry || "No entry for today";
    } else {
      console.log("No puzzle found for today"); 
    }
  }
},

checkWin() {
  if (this.decodedText === this.originalText) {
    this.isSolved = true;
    this.isGiveUp = false;
    this.lockPuzzleForToday(); 
    this.incrementStreak();
  }
},
revealSolution() {
  alert(`The solution is: ${this.originalText}`);
  this.lockPuzzleForToday(); 
  this.incrementStreak();
},
showHint() {
  if (this.hintIndex < this.additionalHints.length) {
    this.currentHintToShow = this.additionalHints[this.hintIndex];
    this.hintIndex++;

    if (this.hintIndex === this.additionalHints.length) {
      this.showGiveUp = true;
    }
  } else {
    this.currentHintToShow = "No more hints available!";
  }
},
lockPuzzleForToday() {
  this.isPuzzleLocked = true;
  localStorage.setItem("puzzleLocked", "true");
  localStorage.setItem("isSolved", this.isSolved ? "true" : "false");
  this.modalContent = this.isSolved ? "You solved it!" : "Try again tomorrow!";
  this.startCountdownToMidnight();
  },
  
giveUp() {
  this.isGiveUp = true;
  this.isSolved = false;
  this.lockPuzzleForToday();
  this.giveUpMessage = "You gave up. Try again next time!";
  localStorage.setItem("isGiveUp", "true");
  localStorage.setItem("isSolved", "false");
  this.modalContent = "Try again tomorrow!";
},
startCountdownToMidnight() {
  const now = new Date();
  console.log("⏳ Current time:", now.toLocaleString());

  const midnight = new Date();
  midnight.setUTCHours(24, 0, 0, 0); 

  console.log("🎯 Target reset time:", midnight.toLocaleString());

  const timeLeft = midnight - now;
  console.log(`⏳ Time until reset: ${Math.floor(timeLeft / 1000)}s`);

  setTimeout(() => {
    console.log("🚀 Midnight hit — resetting!");
    this.resetForNewDay();
  }, timeLeft);
},
incrementStreak() {
  this.currentStreak++;
  localStorage.setItem("currentStreak", this.currentStreak);
  if (this.currentStreak > this.highestStreak) {
    this.highestStreak = this.currentStreak;
    localStorage.setItem("highestStreak", this.highestStreak);
  }
},
resetForNewDay() {
  console.log("🎉 Resetting for new day!");
  localStorage.removeItem("puzzleLocked");
  localStorage.removeItem("isSolved");
  localStorage.removeItem("isGiveUp");
  localStorage.setItem("lastPuzzleDate", new Date().toISOString().split("T")[0]);

  this.isSolved = false;
  this.isGiveUp = false;
  this.isPuzzleLocked = false;
  this.modalContent = "";
  this.newPuzzleReady = true;

  this.guesses = [];
  this.currentHintToShow = "";
  this.hintIndex = 0;

  this.selectPuzzleForToday();
  this.$forceUpdate();
},

resetPuzzle() {
  this.isSolved = false;
  this.isGiveUp = false;
  this.isPuzzleLocked = false;
  this.newPuzzleReady = false;
  this.guesses = [];
  this.currentHintToShow = "";
  this.hintIndex = 0;

  localStorage.removeItem("puzzleLocked");
  localStorage.removeItem("isSolved");
  localStorage.removeItem("isGiveUp");

  this.selectPuzzleForToday();
},
showHint() {
  if (this.hintIndex < this.additionalHints.length) {
    this.currentHintToShow = this.additionalHints[this.hintIndex];
    this.hintIndex++;
  }
},
resetStreak() {
  this.currentStreak = 0;
  localStorage.setItem("currentStreak", this.currentStreak);
},
loadStreaks() {
  this.highestStreak = parseInt(localStorage.getItem("highestStreak")) || 0;
  this.currentStreak = parseInt(localStorage.getItem("currentStreak")) || 0;
  if (!this.isPuzzleSolvedToday() && !this.isPuzzleLocked) {
    this.currentStreak = 0;
  }
},
isPuzzleSolvedToday() {
  const puzzleSolved = localStorage.getItem("isSolved") === "true";
  const isGiveUp = localStorage.getItem("isGiveUp") === "true";
  return puzzleSolved || isGiveUp;
},
generateShareResult() {
    const today = new Date().toLocaleDateString("en-US", { month: "long", day: "numeric", year: "numeric" });
    const hintCount = this.additionalHints.length;
    const hintsUsed = this.hintIndex;

    const progressBar = this.codedPhrase
      .split("")
      .map((char, index) => (char.match(/[A-Z]/) ? (this.guesses[index] ? "🟩" : "⬜") : "⬛"))
      .join("");

    const resultText = `Secret Agent — ${today}\n` +
      `${this.isSolved ? `✅ Puzzle solved` : "❌ Gave up"}\n` +
      `Streak: ${this.currentStreak} days\n` +
      `Hints used: ${hintsUsed}/${hintCount}\n` +
      `${progressBar}`;

    navigator.clipboard.writeText(resultText).then(() => {
      this.showNotification = true;
      this.notificationMessage = "Result copied to clipboard. Share with your friends.";
      
      // Hide the notification after 3 seconds
      setTimeout(() => {
        this.showNotification = false;
      }, 3000);
    });
  },
},
mounted() {
  this.loadPuzzles();

  const lastPuzzleDate = localStorage.getItem("lastPuzzleDate");
  const today = new Date().toISOString().split("T")[0];

  if (lastPuzzleDate !== today) {
    console.log("🌅 New day detected — resetting!");
    this.resetForNewDay();
  }

  if (localStorage.getItem("puzzleLocked") === "true") {
    this.isPuzzleLocked = true;
    this.modalContent = "Puzzle is locked. New one arrives at midnight!";
    this.startCountdownToMidnight();
  }
}
};

  </script>

<style>

* {
    font-family: "Special Elite", system-ui;
}
.cryptoquip-container {
  font-family: "Courier New", Courier, monospace;
  padding: 20px;
}

.letter-container {
    margin-left: 10rem;
    margin-right: 10rem;
    box-shadow: rgba(100, 100, 111, 0.2) 0px 7px 29px 0px;
    padding: 7rem;
    background-color: #fbfefe;
}

.title {
font-size: 30px;
color: #252525;
}

.hello {
    padding-bottom: 3rem;
    color: #252525;
    background-color: #fbfefe;
}

.instructions {
    display: flex;
    justify-content: center;
    text-align: start;
    text-indent: 2rem;
    color: #252525;
    background-color: #fbfefe;
}

.hint {
  font-size: 20px;
  color: #252525;
  margin-bottom: 1px;
  padding-bottom: 3rem;
  padding-top: 3rem;
  text-align: center;
  background-color: #fbfefe;
  align-items: center;
}

p, h2 {
    background-color: #fbfefe;
}

.add-hint {
    background-color: #fbfefe;
}

.puzzle {
  display: flex;
  flex-wrap: wrap;
  gap: .5rem;
  justify-content: center;
  margin-bottom: 1rem;
  margin-top: 3rem;
  background-color: #fbfefe;
}

.puzzle-char {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 40px;
  height: 50px;
  border: 1px ridge #252525;
  background-color: white;
  font-size: 18px;
  text-transform: uppercase;
  background-color: #fbfefe;
  padding: .3rem;
}

.cipher {
  font-size: 18px;
  font-weight: 500;
  color: #444;
  margin-bottom: 5px;
  background-color: white;
  border-bottom: solid 1px black;  
  background-color: #fbfefe;
}

.puzzle-char input {
  width: 25px;
  height: 25px;
  text-align: center;
  font-size: 18px;
  border: none;
  background: transparent;
  outline: none;
  text-transform: uppercase;
}


.non-letter {
  font-size: 20px;
  color: #555;
  margin: 0 5px;
  background-color: #fbfefe;
}

.progress {
  margin-top: 3rem;
  font-size: 40px;
  color: #383737;
  background-color: #fbfefe;
  text-align: center;
}

.streak {
    color: #313030;
    position: absolute;
    top: 0;
    right: 0;
    padding: 1rem;
}

.win-message {
  margin-top: 15px;
  color: #4caf50;
  font-size: 22px;
  font-weight: bold;
  animation: pop 0.5s ease-out;
}

.hint-btn {
  margin-top: 15px;
  padding: 8px 16px;
  background: #3d3d3d;
  color: #fff;
  border: none;
  cursor: pointer;
  width: 30%;
}

.input-box {
  color: #3d3d3d;;
}

.input-box::placeholder {
  color: rgb(173, 11, 11);
  font-weight: 500;
}

.hint-column {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 10px;
  margin-top: 10px;
  background-color: #fbfefe;
  width: 100%; 
}

.give-up {
  background-color: #fbfefe;
  display: flex;
  justify-content: center;
  margin-top: 1rem;
}

.hint-box { 
  background-color: #fbfefe;
  text-align: start;
  width: 80%; 
  box-shadow: rgba(0, 0, 0, 0.05) 0px 0px 0px 1px;
  transition: background-color 0.3s ease;
  padding: .3rem;
  border-radius: .3rem;
}

.hint-box:not(.locked) {
  background-color: #fbfefe;
  color: #1f1e1e;
  font-family: 'Courier New', Courier, monospace;
}

.hint-box.locked {
  background-color: #fbfefe;
  color: #1f1e1e;
  font-family: 'Courier New', Courier, monospace;
}


.hint-btn:hover {
    background: #2D4B73;
}

.give-up-btn {
  background-color: #0A0A0A;
  margin-top: 15px;
  padding: 8px 16px;
  background: #bd2d2d;
  color: #fff;
  border: none;
  cursor: pointer;
}

.give-up-btn:hover {
  background: #c03631;
}

button:disabled {
  background: #999;
  cursor: not-allowed;
}

.additional-hint {
  margin-top: 20px;
  font-size: 18px;
  color: #444;
  background-color: #fbfefe;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal {
  background-color: white;
  padding: 200px;
  border-radius: 8px;
  max-width: 400px;
  width: 100%;
  text-align: center;
}
.share-button {
  background-color: #2D4B73;
  color: #fff;
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  margin-top: 10px;
  transition: background 0.3s ease;
}

.share-button:hover {
  background-color: #3e8e41;
}

.new-puzzle-button{
  background-color: #ffffff;
}

.btn-new-puzzle {
  background-color: #303330; 
  color: white; 
  padding: 10px 20px;
  border-radius: 5px;
  font-size: 16px;
  border: none;
  cursor: pointer;
  transition: background-color 0.3s;
  margin-top: 2rem;
}

.btn-new-puzzle:hover {
  background-color: #2D4B73;
}


@keyframes pop {
  0% {
    transform: scale(0.8);
  }
  100% {
    transform: scale(1);
  }
}
</style>
