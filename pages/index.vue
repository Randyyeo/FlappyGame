<template>
  <div>
    <canvas
      id="gameCanvas"
      width="320"
      height="480"
      style="display: block; margin: 0 auto"
    ></canvas>
  </div>
</template>

<script>
export default {
  name: "FlappyPage",
  head() {
    return {
      // Other meta information
      script: [{ src: "https://code.createjs.com/1.0.0/createjs.min.js" }],
    };
  },
  data() {
    return {
      stage: null,
      loader: null,
      flappy: null,
      status: null,
      polygon: null,
      jumpListener: null,
      pipeCreator: null,
      score: null,
      scoreText: null,
      scoreTextOutline: null
    };
  },
  mounted() {
    this.stage = new createjs.StageGL("gameCanvas");

    createjs.Ticker.timingMode = createjs.Ticker.RAF_SYNCHED;
    createjs.Ticker.framerate = 60;
    createjs.Ticker.addEventListener("tick", this.stage);

    var background = new createjs.Shape();
    background.graphics
      .beginLinearGradientFill(
        ["#2573BB", "#6CB8DA", "#567A32"],
        [0, 0.85, 1],
        0,
        0,
        0,
        480
      )
      .drawRect(0, 0, 320, 480);
    background.x = 0;
    background.y = 0;
    background.name = "background";
    background.cache(0, 0, 320, 480);

    this.stage.addChild(background);

    var manifest = [
      { src: "cloud.png", id: "cloud" },
      { src: "flappy.png", id: "flappy" },
      { src: "pipe.png", id: "pipe" },
    ];

    this.loader = new createjs.LoadQueue(true);
    this.loader.addEventListener("complete", this.handleComplete);
    this.loader.loadManifest(manifest, true, "/");
  },
  methods: {
    handleComplete() {
      this.status = false;
      this.createClouds();
      this.createFlappy();
      this.createScore();
      this.jumpListener = this.stage.on("stagemousedown", this.jumpFlappy);
      createjs.Ticker.addEventListener("tick", this.checkCollision);
      /* this.polygon = new createjs.Shape();
      this.stage.addChild(this.polygon); */
    },
    createClouds() {
      var clouds = [];
      for (var i = 0; i < 3; i++) {
        clouds.push(new createjs.Bitmap(this.loader.getResult("cloud")));
      }

      clouds[0].x = 40;
      clouds[0].y = 20;
      clouds[1].x = 140;
      clouds[1].y = 70;
      clouds[2].x = 100;
      clouds[2].y = 130;

      for (var i = 0; i < 3; i++) {
        var directionMultiplier = i % 2 == 0 ? -1 : 1;
        var originalX = clouds[i].x;
        createjs.Tween.get(clouds[i], { loop: true })
          .to(
            { x: clouds[i].x - 200 * directionMultiplier },
            3000,
            createjs.Ease.getPowInOut(2)
          )
          .to({ x: originalX }, 3000, createjs.Ease.getPowInOut(2));
        this.stage.addChild(clouds[i]);
      }
    },
    createFlappy() {
      this.flappy = new createjs.Bitmap(this.loader.getResult("flappy"));
      this.flappy.regX = this.flappy.image.width / 2;
      this.flappy.regY = this.flappy.image.height / 2;
      this.flappy.x = this.stage.canvas.width / 2;
      this.flappy.y = this.stage.canvas.height / 2;
      this.stage.addChild(this.flappy);
    },
    jumpFlappy() {
      if (!this.status){
        this.startGame()
      }
      createjs.Tween.get(this.flappy, { override: true })
        .to(
          { y: this.flappy.y - 60, rotation: -10 },
          300,
          createjs.Ease.getPowOut(1)
        )
        .to(
          {
            y: this.stage.canvas.height + this.flappy.image.width / 2,
            rotation: 30,
          },
          1200,
          createjs.Ease.getPowIn(2)
        )
        .call(this.gameOver);
    },
    gameOver() {
      createjs.Tween.removeAllTweens();
      this.stage.off("stagemousedown", this.jumpListener)
      clearInterval(this.pipeCreator);
      createjs.Ticker.removeEventListener("tick", this.checkCollision)
      setTimeout(this.stage.on("stagemousedown", this.resetGame, null, true), 2000);
      
    },
    resetGame(){
      
      var childrenToRemove = this.stage.children.filter((child)=>child.name != "background");
      for (var i=0; i<childrenToRemove.length;i++){
        this.stage.removeChild(childrenToRemove[i])
      }
      this.handleComplete()
    },
    startGame(){
      this.status = true;
      this.createPipes();
      this.pipeCreator = setInterval(this.createPipes, 6000);
    },
    createScore(){
      this.score = 0;
      this.scoreText = new createjs.Text(this.score, "bold 48px Arial", "#FFFFFF");
      this.scoreText.textAlign = "center";
      this.scoreText.textBaseline = "middle";
      this.scoreText.x = 40;
      this.scoreText.y = 40;
      var bounds = this.scoreText.getBounds();
      this.scoreText.cache(-40, -40, bounds.width*3 + Math.abs(bounds.x), bounds.height + Math.abs(bounds.y));

      this.scoreTextOutline = this.scoreText.clone();
      this.scoreTextOutline.color = "#000000";
      this.scoreTextOutline.outline = 2;
      bounds = this.scoreTextOutline.getBounds();
      this.scoreTextOutline.cache(-40, -40, bounds.width*3 + Math.abs(bounds.x), bounds.height + Math.abs(bounds.y));
      
      this.stage.addChild(this.scoreText, this.scoreTextOutline);
    },
    incrementScore(){
      this.score++;
      this.scoreText.text = this.scoreTextOutline.text = this.score;
      this.scoreText.updateCache();
      this.scoreTextOutline.updateCache();
    },
    createPipes() {
      var topPipe, bottomPipe;
      var position = Math.floor(Math.random() * 280 + 100);

      topPipe = new createjs.Bitmap(this.loader.getResult("pipe"));
      topPipe.y = position - 75;
      topPipe.x = this.stage.canvas.width + topPipe.image.width / 2;
      topPipe.rotation = 180;
      topPipe.name = "pipe";

      bottomPipe = new createjs.Bitmap(this.loader.getResult("pipe"));
      bottomPipe.y = position + 75;
      bottomPipe.x = this.stage.canvas.width + bottomPipe.image.width / 2;
      bottomPipe.skewY = 180;
      bottomPipe.name = "pipe";

      topPipe.regX = bottomPipe.regX = topPipe.image.width / 2;

      createjs.Tween.get(topPipe)
        .to({ x: 0 - topPipe.image.width }, 10000)
        .call(
          this.removePipe(topPipe)
        )
        .addEventListener("change", this.updatePipe);
      createjs.Tween.get(bottomPipe)
        .to({ x: 0 - bottomPipe.image.width }, 10000)
        .call(
          this.removePipe(bottomPipe)
        );

      var scoreIndex = this.stage.getChildIndex(this.scoreText);

      this.stage.addChildAt(bottomPipe, topPipe, scoreIndex);
    },
    removePipe(pipe) {
      this.stage.removeChild(pipe)
    },
    updatePipe(event){
      var pipeUpdate = event.target.target;
      if ((pipeUpdate.x - pipeUpdate.regX + pipeUpdate.image.width) < (this.flappy.x - this.flappy.regX)){
        event.target.removeEventListener("change", this.updatePipe);
        this.incrementScore();
      }
    },
    checkCollision(){
      var leftX = this.flappy.x - this.flappy.regX + 5; 
      var leftY = this.flappy.y - this.flappy.regY + 5;
      var point = [
        new createjs.Point(leftX, leftY), // Top Left
        new createjs.Point(leftX + this.flappy.image.width - 10, leftY), // Top Right
        new createjs.Point(leftX, leftY + this.flappy.image.height - 10), // Bottom Left
        new createjs.Point(leftX + this.flappy.image.width - 10, leftY + this.flappy.image.height - 10) // Bottom Right
      ];
      // A way to check the points where the game registers
      /* this.polygon.graphics.clear().beginStroke("black");
      this.polygon.graphics.moveTo(point[0].x, point[0].y).lineTo(point[2].x, point[2].y).lineTo(point[3].x, point[3].y)
      .lineTo(point[1].x, point[1].y).lineTo(point[0].x, point[0].y); */

      for (var i = 0; i < point.length; i++){
        var objects = this.stage.getObjectsUnderPoint(point[i].x, point[i].y);
        if (objects.filter((object)=>object.name == "pipe").length > 0){
          this.gameOver();
          return;
        }
      }
    }
  },
};
</script>
