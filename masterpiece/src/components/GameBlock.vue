 <template>
  <div id="gameblock">
    <canvas
      ref="canvas"
      id="up-part"
      @mousedown="mouseDown"
      @mouseup="mouseUp"
      @mouseleave="mouseLeave"
      @mousemove="mouseMove"
    ></canvas>
    <div id="down-part">
      <div id="userblock">
        <div id="leftuser">
          <img src="@/assets/left.png" />
          <br />
          <br />
          <h3>{{leftUserName}}</h3>
          <div>{{isDrawerMessage}}</div>
        </div>
        <div id="rightuser">
          <img src="@/assets/right.png" />
          <br />
          <br />
          <h3>{{rightUserName}}</h3>
        </div>
      </div>
      <div id="room">
        <div id="room-message" value>
          <div>Time: {{guessTime}}</div>
          <div>{{roomMessage}}</div>
        </div>
        <div id="room-form">
          <input type="text" id="room-text" v-model="roomText" @keyup.13="onMessage" />
          <input type="submit" id="room-submit" value="Send" @click="onMessage" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "GameBlock",
  data() {
    return {
      userName: "",
      leftUserName: this.$store.state.userName,
      rightUserName: this.$store.state.opponentName,
      roomText: "",
      roomMessage: "",
      canvas: null,
      canvasContext: null,
      isMouseDown: false,
      currentX: 0,
      currentY: 0,
      guessTime: 0,
      timer: null,
      isDrawerMessage: ""
    };
  },
  methods: {
    onMessage: function() {
      if (!this.$store.state.isDrawer) {
        let message = this.$store.state.userName + ": " + this.roomText + "\n";
        let messageInfo = {
          roomId: this.$store.state.roomId,
          message: message
        };
        this.$socket.emit("onMessage", JSON.stringify(messageInfo));
        this.$socket.emit("guess", this.roomText);
        this.roomMessage += message;
        this.roomText = "";
      }
    },
    mouseDown: function(e) {
      if (this.$store.state.isDrawer) {
        this.isMouseDown = true;
        this.currentX = e.pageX - this.canvas.offsetLeft;
        this.currentY = e.pageY - this.canvas.offsetTop;
        let point = {
          x: this.currentX,
          y: this.currentY
        };
        this.$socket.emit("mouseDown", JSON.stringify(point));
      }
    },
    mouseUp: function() {
      if (this.$store.state.isDrawer) {
        this.isMouseDown = false;
        this.$socket.emit("mouseUp");
      }
    },
    mouseLeave: function() {
      if (this.$store.state.isDrawer) {
      this.isMouseDown = false;
      this.$socket.emit("mouseLeave");
      }
    },
    mouseMove: function(e) {
      // 這邊有確認滑鼠點擊不錯，但沒處理超出canvas框框的時候怎辦
      if (this.isMouseDown && this.$store.state.isDrawer) {
        this.canvasContext.beginPath();
        this.canvasContext.lineJoin = "round";
        this.canvasContext.moveTo(this.currentX, this.currentY);
        this.canvasContext.lineTo(e.pageX - this.canvas.offsetLeft, e.pageY - this.canvas.offsetTop);
        this.canvasContext.stroke();
        this.canvasContext.closePath();
        this.currentX = e.pageX - this.canvas.offsetLeft;
        this.currentY = e.pageY - this.canvas.offsetTop;
        let point = {
          x: this.currentX,
          y: this.currentY
        };
        this.$socket.emit("mouseMove", JSON.stringify(point));
      }
    }
  },
  sockets: {
    onMessage: function(value) {
      this.roomMessage += value;
    },
    mouseDown: function(point) {
      point = JSON.parse(point);
      this.isMouseDown = true;
      this.currentX = point.x;
      this.currentY = point.y;
    },
    mouseUp: function() {
      this.isMouseDown = false;
    },
    mouseLeave: function() {
      this.isMouseDown = false;
    },
    mouseMove: function(point) {
      point = JSON.parse(point);
      if (this.isMouseDown) {
        this.canvasContext.beginPath();
        this.canvasContext.lineJoin = "round";
        this.canvasContext.moveTo(this.currentX, this.currentY);
        this.canvasContext.lineTo(point.x, point.y);
        this.canvasContext.stroke();
        this.canvasContext.closePath();
        this.currentX = point.x;
        this.currentY = point.y;
      }
    },
    gameStart: function(timestamp) {
      this.canvasContext.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.guessTime = Math.floor((60000 - (Date.now() - timestamp)) / 1000);
      let that = this;
      this.timer = setInterval(() => {
        if (that.guessTime != 0) {
          that.guessTime--;
        } else {
          clearInterval(that.timer);
          that.$socket.emit("gameEnd", that.$store.state.roomId);
        }
      }, 1000);
    },
    gameEnd: function(answer) {
      clearInterval(this.timer);
      this.isDrawerMessage = "";
      this.guessTime = 0;
      this.$store.commit("setIsDrawer", false);
      this.canvasContext.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.canvasContext.fillText("The answer is " + answer, 100, 60);
      let that = this;
      setTimeout(function() {
        that.$socket.emit("getReady", that.$store.state.roomId);
      }, 3000);
    },
    isDrawer: function() {
      this.$store.commit("setIsDrawer", true);
      this.isDrawerMessage = "Is Your Turn To Draw";
    },
    correctGuess: function() {
      this.roomMessage += "Congratulation！ You get the correct answer！\n";
    },
    wholeGameEnd: function(answer) {
      clearInterval(this.timer);
      this.isDrawerMessage = "";
      this.canvasContext.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.canvasContext.fillText("The answer is " + answer, 100, 60);
      this.canvasContext.fillText("Game End", 100, 80);
      let that = this;
      setTimeout(function() {
        that.$router.push("/login");
      }, 5000);
    }
  },
  mounted() {
    this.canvas = this.$refs.canvas;
    let ctx = this.canvas.getContext("2d");
    this.canvasContext = ctx;
    this.$socket.emit("getReady", this.$store.state.roomId);
  }
};
</script>

<style scoped>
#up-part {
  width: 99%;
  height: 48vh;
  margin: 0.25%;
  border-style: solid;
  border-width: 1.5px;
  border-radius: 10px;
}
#down-part {
  margin: 0.25%;
  height: 44vh;
}
#userblock {
  width: 40%;
  height: 100%;
  float: left;
  border-style: solid;
  border-width: 1.5px;
  border-radius: 10px;
}
#leftuser {
  float: left;
  width: 49.5%;
  height: 100%;
  border-right-style: dotted;
  text-align: center;
}
#rightuser {
  float: left;
  width: 49.5%;
  height: 100%;
  text-align: center;
}
#room {
  width: 59%;
  height: 100%;
  margin-left: 0.5%;
  float: left;
  border-style: solid;
  border-width: 1.5px;
  border-radius: 10px;
}
#room-message {
  width: 99%;
  height: 86%;
  margin: 0.8%;
  white-space: pre-line;
  overflow: auto;
  font-size: 16px;
  line-height: 24px;
}
#room-form {
  width: 99%;
  height: 6%;
  margin: 0.5%;
}
#room-text {
  width: 90%;
  height: 100%;
  margin: 0.5%;
  border-style: solid;
  border-width: 1px;
  border-radius: 4px;
}
#room-submit {
  width: 8%;
  height: 100%;
  border-style: solid;
  border-width: 1px;
  border-radius: 4px;
}
</style>