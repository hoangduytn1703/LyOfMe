<script setup>
import { ref, onMounted } from "vue";
import Button from './components/Button.vue'
const isShow = ref(false)
const showHeart = (val) => {
  isShow.value = val
}
const showTime = () => {
  document.getElementById("currentTime").innerHTML = new Date().toUTCString();
};
const runFunc = () => {
  var settings = {
    particles: {
      length: 500, // maximum amount of particles

      duration: 2, // particle duration in sec

      velocity: 100, // particle velocity in pixels/sec

      effect: -0.75, // play with this for a nice effect

      size: 30, // particle size in pixels
    },
  };

  /*
  
  * RequestAnimationFrame polyfill by Erik Möller
  
  */

  (function () {
    var b = 0;
    var c = ["ms", "moz", "webkit", "o"];
    for (var a = 0; a < c.length && !window.requestAnimationFrame; ++a) {
      window.requestAnimationFrame = window[c[a] + "RequestAnimationFrame"];
      window.cancelAnimationFrame =
        window[c[a] + "CancelAnimationFrame"] ||
        window[c[a] + "CancelRequestAnimationFrame"];
    }
    if (!window.requestAnimationFrame) {
      window.requestAnimationFrame = function (h, e) {
        var d = new Date().getTime();
        var f = Math.max(0, 16 - (d - b));
        var g = window.setTimeout(function () {
          h(d + f);
        }, f);
        b = d + f;
        return g;
      };
    }
    if (!window.cancelAnimationFrame) {
      window.cancelAnimationFrame = function (d) {
        clearTimeout(d);
      };
    }
  })();

  /*
  
  * Point class
  
  */

  var Point = (function () {
    function Point(x, y) {
      this.x = typeof x !== "undefined" ? x : 0;

      this.y = typeof y !== "undefined" ? y : 0;
    }

    Point.prototype.clone = function () {
      return new Point(this.x, this.y);
    };

    Point.prototype.length = function (length) {
      if (typeof length == "undefined")
        return Math.sqrt(this.x * this.x + this.y * this.y);

      this.normalize();

      this.x *= length;

      this.y *= length;

      return this;
    };

    Point.prototype.normalize = function () {
      var length = this.length();

      this.x /= length;

      this.y /= length;

      return this;
    };

    return Point;
  })();

  /*
  
  * Particle class
  
  */

  var Particle = (function () {
    function Particle() {
      this.position = new Point();

      this.velocity = new Point();

      this.acceleration = new Point();

      this.age = 0;
    }

    Particle.prototype.initialize = function (x, y, dx, dy) {
      this.position.x = x;

      this.position.y = y;

      this.velocity.x = dx;

      this.velocity.y = dy;

      this.acceleration.x = dx * settings.particles.effect;

      this.acceleration.y = dy * settings.particles.effect;

      this.age = 0;
    };

    Particle.prototype.update = function (deltaTime) {
      this.position.x += this.velocity.x * deltaTime;

      this.position.y += this.velocity.y * deltaTime;

      this.velocity.x += this.acceleration.x * deltaTime;

      this.velocity.y += this.acceleration.y * deltaTime;

      this.age += deltaTime;
    };

    Particle.prototype.draw = function (context, image) {
      function ease(t) {
        return --t * t * t + 1;
      }

      var size = image.width * ease(this.age / settings.particles.duration);

      context.globalAlpha = 1 - this.age / settings.particles.duration;

      context.drawImage(
        image,
        this.position.x - size / 2,
        this.position.y - size / 2,
        size,
        size
      );
    };

    return Particle;
  })();

  /*
  
  * ParticlePool class
  
  */

  var ParticlePool = (function () {
    var particles,
      firstActive = 0,
      firstFree = 0,
      duration = settings.particles.duration;
    function ParticlePool(length) {
      // create and populate particle pool
      particles = new Array(length);
      for (var i = 0; i < particles.length; i++) particles[i] = new Particle();
    }
    ParticlePool.prototype.add = function (x, y, dx, dy) {
      particles[firstFree].initialize(x, y, dx, dy);
      // handle circular queue
      firstFree++;
      if (firstFree == particles.length) firstFree = 0;
      if (firstActive == firstFree) firstActive++;
      if (firstActive == particles.length) firstActive = 0;
    };

    ParticlePool.prototype.update = function (deltaTime) {
      var i;
      // update active particles
      if (firstActive < firstFree) {
        for (let i = firstActive; i < firstFree; i++)
          particles[i].update(deltaTime);
      }

      if (firstFree < firstActive) {
        for (let i = firstActive; i < particles.length; i++)
          particles[i].update(deltaTime);
        for (let i = 0; i < firstFree; i++) particles[i].update(deltaTime);
      }

      // remove inactive particles
      while (
        particles[firstActive].age >= duration &&
        firstActive != firstFree
      ) {
        firstActive++;
        if (firstActive == particles.length) firstActive = 0;
      }
    };
    ParticlePool.prototype.draw = function (context, image) {
      // draw active particles
      if (firstActive < firstFree) {
        for (let i = firstActive; i < firstFree; i++)
          particles[i].draw(context, image);
      }
      if (firstFree < firstActive) {
        for (let i = firstActive; i < particles.length; i++)
          particles[i].draw(context, image);
        for (let i = 0; i < firstFree; i++) particles[i].draw(context, image);
      }
    };
    return ParticlePool;
  })();
  /*
   * Putting it all together
   */

  (function (canvas) {
    var context = canvas.getContext("2d"),
      particles = new ParticlePool(settings.particles.length),
      particleRate = settings.particles.length / settings.particles.duration, // particles/sec
      time;
    // get point on heart with -PI <= t <= PI
    function pointOnHeart(t) {
      return new Point(
        160 * Math.pow(Math.sin(t), 3),
        130 * Math.cos(t) -
        50 * Math.cos(2 * t) -
        20 * Math.cos(3 * t) -
        10 * Math.cos(4 * t) +
        25
      );
    }
    // creating the particle image using a dummy canvas

    var image = (function () {
      var canvas = document.createElement("canvas"),
        context = canvas.getContext("2d");
      canvas.width = settings.particles.size;
      canvas.height = settings.particles.size;
      // helper function to create the path
      function to(t) {
        var point = pointOnHeart(t);
        point.x =
          settings.particles.size / 2 +
          (point.x * settings.particles.size) / 350;
        point.y =
          settings.particles.size / 2 -
          (point.y * settings.particles.size) / 350;
        return point;
      }
      // create the path
      context.beginPath();
      var t = -Math.PI;
      var point = to(t);
      context.moveTo(point.x, point.y);
      while (t < Math.PI) {
        t += 0.01; // baby steps!
        point = to(t);
        context.lineTo(point.x, point.y);
      }
      context.closePath();
      // create the fill
      context.fillStyle = "#ea80b0";
      context.fill();
      // create the image
      var image = new Image();
      image.src = canvas.toDataURL();
      return image;
    })();
    // render that thing!

    function render() {
      // next animation frame
      requestAnimationFrame(render);
      // update time
      var newTime = new Date().getTime() / 1000,
        deltaTime = newTime - (time || newTime);
      time = newTime;

      // clear canvas
      context.clearRect(0, 0, canvas.width, canvas.height);
      // create new particles
      var amount = particleRate * deltaTime;
      for (var i = 0; i < amount; i++) {
        var pos = pointOnHeart(Math.PI - 2 * Math.PI * Math.random());
        var dir = pos.clone().length(settings.particles.velocity);
        particles.add(
          canvas.width / 2 + pos.x,
          canvas.height / 2 - pos.y,
          dir.x,
          -dir.y
        );
      }

      // update and draw particles
      particles.update(deltaTime);
      particles.draw(context, image);
    }

    // handle (re-)sizing of the canvas

    function onResize() {
      canvas.width = canvas.clientWidth;

      canvas.height = canvas.clientHeight;
    }

    window.onresize = onResize;

    // delay rendering bootstrap

    setTimeout(function () {
      onResize();

      render();
    }, 10);
  })(document.getElementById("pinkboard"));
};
onMounted(() => {
  runFunc();
  showTime();
  setInterval(function () {
    showTime();
  }, 1000);
});
</script>

<template>
  <canvas id="pinkboard" :class="{ 'show-heart': isShow }"></canvas>
  <div class="text" :class="{'animated animatedFadeInUp fadeInUp': isShow}">
    I
    <svg class="heart" viewBox="0 0 32 29.6">
      <path d="M23.6,0c-3.4,0-6.3,2.7-7.6,5.6C14.7,2.7,11.8,0,8.4,0C3.8,0,0,3.8,0,8.4c0,9.4,9.5,11.9,16,21.2
    c6.1-9.3,16-12.1,16-21.2C32,3.8,28.2,0,23.6,0z" />
    </svg>
    You
  </div>

  <div :class="{ 'hide-button': isShow }">
    <Button @showHeart="showHeart" :disabled="isShow" />
  </div>
</template>

<style lang="scss" scoped>
.show-heart {
  opacity: 1;
}

.hide-button {
  opacity: 0;
}

div {
  opacity: 1;
  transition: all ease-in-out 3s;
}

html,
body {
  height: 100%;
  padding: 0;
  margin: 0;
  background: #000;
}

canvas {
  position: absolute;
  width: 100%;
  height: 100%;
  opacity: 0;
  transition: all ease-in-out 5s;
}

body {
  padding: 25px;
}

.title {
  color: #5c6ac4;
}

/* button */
/* === === CSS RESET === === */
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  list-style: none;
  outline: none;
  border: none;
  text-decoration: none;
  text-shadow: none;
}

/* === === CUSTOM VARIABLES === === */
:root {
  --color-neon: hsl(189 100% 50%);
  --color-alpha: hsl(189 100% 50%/0.8);
  --color-bg: hsl(243, 100%, 15%);
}

/* === === CSS STYLING === === */
body {
  height: 100vh;
  display: grid;
  place-items: center;
  background-color: var(--color-bg);
}

.neon__btn {
  color: var(--color-neon);
  font-size: 3rem;
  font-family: "D3 Euronism";
  border: var(--color-neon) 0.15em solid;
  border-radius: 0.25em;
  padding: 0.25em 1em;
  position: relative;

  /* => neon/glow effect on text + box around */
  text-shadow: 0 0 0.125em var(--color-alpha), 0 0 0.1em var(--color-neon);
  box-shadow: inset 0 0 0.5em var(--color-neon), 0 0 0.85em var(--color-neon);
  /* => "inset" drops box-shadow (glow effect in this case) inside the element */
}

/* => space effect with ground element below btn */
.neon__btn::before {
  content: "";
  position: absolute;
  background-color: var(--color-neon);
  top: 120%;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  /* => to prevent triggering pointer outside btn */

  /* => blur and transform ground element */
  transform: perspective(1em) rotateX(40deg) scale(1, 0.35);
  filter: blur(1em);
  opacity: 0.5;
  transition: all 350ms ease-out;
}

/* => transition effect ground element when hovering over btn */
.neon__btn:hover::before {
  transform: perspective(0.9em) rotateX(45deg) scale(1, 0.35);
  filter: blur(1em);
  opacity: 0.9;
  transition: all 220ms ease-in;
}

/* => btn background when hovering over btn */
.neon__btn:hover {
  background-color: var(--color-neon);
  color: var(--color-bg);
}

/* => outer glow btn */
.neon__btn::after {
  content: "";
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  box-shadow: 0 0 2em 0.5em var(--color-neon);
  background-color: var(--color-neon);
  transition: opacity 100ms linear;
  opacity: 0;
  z-index: -1;
}

/* => outer glow btn when hovering over btn */
.neon__btn:hover::after {
  opacity: 1;
}
.text{
  opacity: 0;
  font-family: 'Archivo Black', sans-serif;
  font-size: 3em;
  margin-top: 1em;
  text-align: center;
  text-transform: uppercase;
  position: absolute;
  bottom: 105px;
  left: 48%;
  transform: translateX(-50px);
  .heart {
    fill: red;
    position: relative;
    top: 5px;
    width: 50px;
    animation: pulse 1s ease infinite,
}
}


@keyframes pulse {
  0% {
    transform: scale(1);
  }

  50% {
    transform: scale(1.3);
  }

  100% {
    transform: scale(1);
  }
}
@keyframes fadeInUp {
    from {
      opacity: 0;
      transform:translateX(-50px) translate3d(0,40px,0)
    }

    to {
      transform:translateX(-50px) translate3d(0,0,0);
      opacity: 1
    }
}

@-webkit-keyframes fadeInUp {
    from {
      opacity: 0;
      transform:translateX(-50px) translate3d(0,40px,0)
    }

    to {
      transform:translateX(-50px) translate3d(0,0,0);
      opacity: 1
    }
}

.animated {
    animation-duration: 3s;
    animation-fill-mode: both;
    -webkit-animation-duration: 3s;
    -webkit-animation-fill-mode: both
}

.animatedFadeInUp {
    opacity: 0
}

.fadeInUp {
    opacity: 0;
    animation-name: fadeInUp;
    -webkit-animation-name: fadeInUp;
}
</style>
