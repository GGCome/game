<script setup>
import { onMounted } from "vue";
import Phaser from "phaser";
import { ethers } from "ethers";

class GameScene extends Phaser.Scene {
  constructor() {
    super("GameScene");
  }

  init() {
    this.player = null;
    this.blocks = null;
    this.cursors = null;
    this.score = 0;
    this.lives = 3;
    this.gameOver = false;
    this.blockData = [];
  }

  preload() {
    this.load.image("player", "/assets/start.jpeg");
    this.load.image("block", "/assets/stop.jpeg");
    this.load.image("powerup", "/assets/good.jpeg");
  }

  create() {
    this.player = this.physics.add.sprite(400, 550, "player");
    this.player.setCollideWorldBounds(true);

    this.blocks = this.physics.add.group();

    this.powerUps = this.physics.add.group();

    this.cursors = this.input.keyboard.createCursorKeys();

    this.scoreText = this.add.text(16, 16, `Score: 0`, {
      fontSize: "20px",
      fill: "#fff",
    });
    this.livesText = this.add.text(16, 40, `Lives: 3`, {
      fontSize: "20px",
      fill: "#fff",
    });

    // Collisions
    this.physics.add.collider(
      this.player,
      this.blocks,
      this.handleBlockCollision,
      null,
      this
    );
    this.physics.add.overlap(
      this.player,
      this.powerUps,
      this.handlePowerUp,
      null,
      this
    );

    this.setupBlockchainListener();
  }

  setupBlockchainListener() {
    const provider = new ethers.JsonRpcProvider(
      "https://linea-sepolia.infura.io/v3/9e65ded734a54b6184a6aa9c1c85f431"
    );

    provider.on("block", async (blockNumber) => {
      try {
        console.log(`New block detected: ${blockNumber}`);
        const block = await provider.getBlock(blockNumber);
        for (const txHash of block.transactions) {
          const tx = await provider.getTransaction(txHash);
          if (tx && tx.gasLimit) {
            const blockSize = Math.min(tx.gasLimit.toNumber() / 1000, 50);
            console.log(
              `Transaction Hash: ${tx.hash}, Block Size: ${blockSize}`
            );
            this.blockData.push({ size: blockSize, hash: tx.hash });
          }
        }
      } catch (error) {
        console.error("Error", error);
      }
    });
  }

  spawnBlock() {
    if (this.blockData.length > 0) {
      const blockInfo = this.blockData.shift();
      const block = this.blocks.create(
        Phaser.Math.Between(50, 750),
        0,
        "block"
      );
      block.displayWidth = block.displayHeight = blockInfo.size;
      block.hash = blockInfo.hash;
      block.setVelocityY(Phaser.Math.Between(100, 300));
    }
  }

  handleBlockCollision(player, block) {
    block.destroy();
    this.lives -= 1;
    this.livesText.setText(`Lives: ${this.lives}`);
    if (this.lives <= 0) {
      this.scene.restart();
    }
  }

  handlePowerUp(player, powerUp) {
    powerUp.destroy();
    this.score += 50; // Example power-up effect
    this.scoreText.setText(`Score: ${this.score}`);
  }

  update() {
    if (this.cursors.left.isDown) {
      this.player.setVelocityX(-200);
    } else if (this.cursors.right.isDown) {
      this.player.setVelocityX(200);
    } else {
      this.player.setVelocityX(0);
    }

    if (this.cursors.up.isDown && this.player.body.touching.down) {
      this.player.setVelocityY(-500);
    }

    if (Phaser.Math.Between(0, 100) > 98) {
      this.spawnBlock();
    }

    this.score += 1 / 60;
    this.scoreText.setText(`Score: ${Math.floor(this.score)}`);
  }
}

const config = {
  type: Phaser.AUTO,
  width: "100%",
  height: "100%",
  physics: {
    default: "arcade",
    arcade: {
      gravity: { y: 300 },
      debug: false,
    },
  },
  scene: [GameScene],
};
onMounted(() => {
  new Phaser.Game(config);
});
</script>
