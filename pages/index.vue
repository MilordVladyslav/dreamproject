<template>
  <div>
    <client-only>
<div id="curtains-canvas"></div>

<div id="slideshow">
  
  <button id="previous-slide" class="slideshow-button">&#x279C;</button>
  <button id="next-slide" class="slideshow-button">&#x279C;</button>
  
  <div id="slides-list">
    <div class="slide">
      <h2 class="slide-title" data-title="Hong Kong">Hong Kong</h2>
      <img src="https://source.unsplash.com/4hluhoRJokI/1280x720" alt="Photo by Simon Zhu on Unsplash" crossorigin />
    </div>
    <div class="slide">
      <h2 class="slide-title" data-title="Chicago">Chicago</h2>
      <img src="https://source.unsplash.com/waedcXSIAKk/1280x720" alt="Photo by Pedro Lastra on Unsplash" crossorigin />
    </div>
    <div class="slide">
      <h2 class="slide-title" data-title="Shanghai">Shanghai</h2>
      <img src="https://source.unsplash.com/D8iZPlX-2fs/1280x720" alt="Photo by Denys Nevozhai on Unsplash" crossorigin />
    </div>
    <div class="slide">
      <h2 class="slide-title" data-title="New York">New York</h2>
      <img src="https://source.unsplash.com/y9WmMWaiftc/1280x720" alt="Photo by Pedro Lastra on Unsplash" crossorigin />
    </div>
  </div>
  
</div>
    </client-only>
  </div>
</template>

<script>
export default {
  head() {
    return {
      script: [
        {
          src: 'https://cdnjs.cloudflare.com/ajax/libs/three.js/110/three.min.js',
        },
        {
          src: 'https://cdnjs.cloudflare.com/ajax/libs/gsap/3.0.5/gsap.min.js',
        },
        {
          src: 'https://cdnjs.cloudflare.com/ajax/libs/jquery/3.4.1/jquery.min.js',
        },
        {
          src: 'https://www.curtainsjs.com/build/curtains.min.js',
        },
        {
          src: 'https://cdnjs.cloudflare.com/ajax/libs/gsap/2.1.3/TweenMax.min.js'
        },
        // {
        //   src: 'https://raw.githubusercontent.com/MilordVladyslav/flat-surface-shader/master/deploy/fss.min.js'
        // }
      ]
    }

  },
  data() {
    return {
      currentPage: ''
    }
  },
  updated() {
    class WebGLSlideshow {
  
  constructor(options = {}) {
    // our options
    this.options = {
      // our webgl container
      webGLContainer: options.slideshow || document.getElementById("curtains-canvas"),
      
      // slideshow state and values
      // our main slideshow wrapper
      slideshow: options.slideshow || document.getElementById("slides-list"),
      // our slides
      slides: options.slides || document.getElementsByClassName("slide"),
      // our nav buttons
      buttons: {
        prevButton: options.prevButton || document.getElementById("previous-slide"),
        nextButton: options.nextButton || document.getElementById("next-slide"),
      },
      
      // duration of the slide change animation
      duration: options.duration || 1,
    };
    
    this.activeTextureIndex = 0;
    this.nextTextureIndex = 1;
    this.nbSlides = this.options.slides.length,
      
    this.isAnimating = false;
    
    this.init();
  }
  
  init() {
    // set up the webgl context
    this.setupCurtains();
    // add the button planes
    this.setupButtonPlanes();
    // add the slideshow plane
    this.setupSlideshowPlane();
    
    // init navigation
    this.initNavigation();
  }
  
  setupCurtains() {
    // create a new Curtains object
    this.curtains = new Curtains({
      container: this.options.webGLContainer
    }).onError( () => {
      // TODO handle webgl errors
    });
  }
  
  setupSlideshowPlane() {
    const vs = `
      #ifdef GL_ES
      precision mediump float;
      #endif

      // default mandatory variables
      attribute vec3 aVertexPosition;
      attribute vec2 aTextureCoord;

      uniform mat4 uMVMatrix;
      uniform mat4 uPMatrix;

      // varyings : notice we've got 2 texture coords varyings
      // one for our visible texture
      // and one for the upcoming texture
      varying vec3 vVertexPosition;
      varying vec2 vTextureCoord;
      varying vec2 vActiveTextureCoord;
      varying vec2 vNextTextureCoord;

      // textures matrices
      uniform mat4 activeTexMatrix;
      uniform mat4 nextTexMatrix;

      // custom uniforms
      uniform float uTransition;

      void main() {
        gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);

        // varyings
        vTextureCoord = aTextureCoord;
        vActiveTextureCoord = (activeTexMatrix * vec4(aTextureCoord, 0.0, 1.0)).xy;
        vNextTextureCoord = (nextTexMatrix * vec4(aTextureCoord, 0.0, 1.0)).xy;
        vVertexPosition = aVertexPosition;
      }
    `;

    const fs = `
      #ifdef GL_ES
      precision mediump float;
      #endif

      varying vec3 vVertexPosition;
      varying vec2 vTextureCoord;
      varying vec2 vActiveTextureCoord;
      varying vec2 vNextTextureCoord;

      // custom uniforms
      uniform float uTransition;
      uniform float uButtonPos;

      // our textures samplers
      // notice how it matches the sampler attributes of the textures we created dynamically
      uniform sampler2D activeTex;
      uniform sampler2D nextTex;

      // Simplex 2D noise
      //
      vec3 permute(vec3 x) {
        return mod(((x*34.0)+1.0)*x, 289.0);
      }

      float snoise(vec2 v){
        const vec4 C = vec4(0.211324865405187, 0.366025403784439, -0.577350269189626, 0.024390243902439);
        vec2 i  = floor(v + dot(v, C.yy) );
        vec2 x0 = v -   i + dot(i, C.xx);
        vec2 i1;
        i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
        vec4 x12 = x0.xyxy + C.xxzz;
        x12.xy -= i1;
        i = mod(i, 289.0);
        vec3 p = permute( permute( i.y + vec3(0.0, i1.y, 1.0 ))
        + i.x + vec3(0.0, i1.x, 1.0 ));
        vec3 m = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy),
        dot(x12.zw,x12.zw)), 0.0);
        m = m*m ;
        m = m*m ;
        vec3 x = 2.0 * fract(p * C.www) - 1.0;
        vec3 h = abs(x) - 0.5;
        vec3 ox = floor(x + 0.5);
        vec3 a0 = x - ox;
        m *= 1.79284291400159 - 0.85373472095314 * ( a0*a0 + h*h );
        vec3 g;
        g.x  = a0.x  * x0.x  + h.x  * x0.y;
        g.yz = a0.yz * x12.xz + h.yz * x12.yw;
        return 130.0 * dot(m, g);
      }

      void main() {
        vec4 noise = vec4(vec3(snoise(vTextureCoord * sqrt(2.0))), 1.0);

        float distanceFromCenter = distance(vTextureCoord, vec2(uButtonPos, 0.5)) * 0.9;

        // calculate an effect that goes from 0 to 1 depenging on uOpacity and distanceToLeft
        float spreadFromCenter = clamp((uTransition * (1.0 - distanceFromCenter) - 1.0) + uTransition * 2.0, 0.0, 1.0);

        vec4 firstImage = texture2D(activeTex, vActiveTextureCoord + noise.r * spreadFromCenter * 0.175);
        vec4 secondImage = texture2D(nextTex, vNextTextureCoord - noise.r * (1.0 - spreadFromCenter) * 0.175);

        // mix both texture
        vec4 finalImage = mix(firstImage, secondImage, spreadFromCenter);

        // handling premultiplied alpha
        finalImage = vec4(finalImage.rgb * finalImage.a, finalImage.a);

        gl_FragColor = finalImage;
      }
    `;
    
    // add the slideshow plane
    this.slideshowPlane = this.curtains.addPlane(this.options.slideshow, {
      vertexShader: vs,
      fragmentShader: fs,
      uniforms: {
        transition: {
          name: "uTransition",
          type: "1f",
          value: 0,
        },
        buttonPos: {
          name: "uButtonPos",
          type: "1f",
          value: 0,
        },
      }
    });
    
    if(this.slideshowPlane) {

      this.slideshowPlane.onReady( () => {
        // the idea here is to create two additionnal textures
        // the first one will contain our visible image
        // the second one will contain our entering (next) image
        // that way we will deal with only activeTex and nextTex samplers in the fragment shader
        // and have as many images in the slideshow as we want...
        
        this.slideshowPlane.userData = {
          // we set our very first image as the active texture
          activeTex: this.slideshowPlane.createTexture({
            sampler: "activeTex",
            fromTexture: this.slideshowPlane.textures[this.activeTextureIndex]
          }),
          // we set the second image as next texture but this is not mandatory
          // as we will reset the next texture on slide change
          nextTex: this.slideshowPlane.createTexture({
            sampler: "nextTex",
            fromTexture: this.slideshowPlane.textures[this.nextTextureIndex]
          })
        };
      });
    }
  }
  
  initNavigation() {
    // show first slide title
    //this.options.slides[this.activeTextureIndex].classList.add("slide--active");
    TweenMax.to(this.options.slides[this.activeTextureIndex].querySelector(".slide-title"), this.options.duration / 2, {
      ease: Power2.easeIn,
      opacity: 1,
      scaleX: 1,
      scaleY: 1,
      force3D: true,
    }).delay(this.options.duration / 2);
    
    // going to next image
    this.options.buttons.nextButton.addEventListener("click", () => {
      if (!this.isAnimating) {
        this.isAnimating = true;

        // check what will be next image
        if (this.activeTextureIndex < this.nbSlides - 1) {
          this.nextTextureIndex = this.activeTextureIndex + 1;
        }
        else {
          this.nextTextureIndex = 0;
        }
        
        // apply it to our next texture
        this.slideshowPlane.userData.nextTex.setSource(this.slideshowPlane.images[this.nextTextureIndex]);

        // change button pos uniform
        this.slideshowPlane.uniforms.buttonPos.value = 1;
        
        // launch tween
        this.changeSlide();
      }
    });
    
    // going to previous image
    this.options.buttons.prevButton.addEventListener("click", () => {
      if (!this.isAnimating) {
        this.isAnimating = true;

        // check what will be next image
        if (this.activeTextureIndex === 0) {
          this.nextTextureIndex = this.nbSlides - 1;
        }
        else {
          this.nextTextureIndex = this.activeTextureIndex - 1;
        }
        
        // apply it to our next texture
        this.slideshowPlane.userData.nextTex.setSource(this.slideshowPlane.images[this.nextTextureIndex]);

        // change button pos uniform
        this.slideshowPlane.uniforms.buttonPos.value = 0;
        
        // launch tween
        this.changeSlide();
      }
    });
  }
  
  changeSlide() {
    this.changeTitles();
    
    TweenMax.to(this.slideshowPlane.uniforms.transition, this.options.duration, {
      value: 1,
      ease: Power2.easeOut,
      onStart: () => {
        this.options.slides[this.activeTextureIndex].classList.remove("slide--active");
      },
      onComplete: () => {
        // we're done animating our slideshow
        this.isAnimating = false;
        this.activeTextureIndex = this.nextTextureIndex;
        // our next texture becomes our active texture
        this.slideshowPlane.userData.activeTex.setSource(this.slideshowPlane.images[this.activeTextureIndex]);
        // reset transition value
        this.slideshowPlane.uniforms.transition.value = 0;
        
        // show active slide title
        //this.options.slides[this.activeTextureIndex].classList.add("slide--active");
      }
    });
  }
  
  changeTitles() {
    const activeTitle = this.options.slides[this.activeTextureIndex].querySelector(".slide-title");
    const nextTitle = this.options.slides[this.nextTextureIndex].querySelector(".slide-title");
    
    TweenMax.to(activeTitle, this.options.duration / 2, {
      ease: Power2.easeIn,
      opacity: 0,
      scaleX: 1.15,
      scaleY: 1.15,
      force3D: true,
    });
    
    TweenMax.to(nextTitle, this.options.duration / 2, {
      ease: Power2.easeOut,
      opacity: 1,
      scaleX: 1,
      scaleY: 1,
      force3D: true,
    }).delay(this.options.duration / 2);
  }
  
  setupButtonPlanes() {
    const buttonVs = `
      #ifdef GL_ES
      precision mediump float;
      #endif
    
      // default mandatory variables
      attribute vec3 aVertexPosition;
      attribute vec2 aTextureCoord;
    
      uniform mat4 uMVMatrix;
      uniform mat4 uPMatrix;
    
      varying vec3 vVertexPosition;
      varying vec2 vTextureCoord;
    
      // custom uniforms
      uniform float uTime;
    
      void main() {
        gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
    
        // varyings
        vTextureCoord = aTextureCoord;
        vVertexPosition = aVertexPosition;
      }
    `;

    const buttonFs = `
      #ifdef GL_ES
      precision mediump float;
      #endif
    
      varying vec3 vVertexPosition;
      varying vec2 vTextureCoord;
    
      // custom uniforms
      uniform float uTime;
      uniform float uHoverEffect;
    
      void main() {
        vec4 color;
        float circleRadius = mix(0.7, 0.8, uHoverEffect);
        float distortionEffect = mix(0.025, 0.05, uHoverEffect);
        vec2 distortedTextCoords = vec2(vTextureCoord.x + sin(uTime / 30.0) * cos(vTextureCoord.y * 5.0) * distortionEffect, vTextureCoord.y + cos(uTime / 30.0) * sin(vTextureCoord.x * 5.0) * distortionEffect);
    
        float hole = step(circleRadius, distance(distortedTextCoords, vec2(0.5, 0.5)) * 2.0);
        vec3 circle = vec3(1.0 - hole);

        // red button
        circle.r *= 0.95;
        circle.g *= 0.1;
        circle.b *= 0.2;
    
        float opacity = 0.75 + (1.0 - uHoverEffect) * 0.25;
    
        color = vec4(circle * opacity, step(0.5, circle.r) * opacity);
    
        gl_FragColor = color;
      }
    `;
    
    const buttonParams = {
      vertexShader: buttonVs,
      fragmentShader: buttonFs,
      transparent: true, // important so they're drawn on top of the slideshow plane
      uniforms: {
        time: {
          name: "uTime",
          type: "1f",
          value: 0,
        },
        hoverEffect: {
          name: "uHoverEffect",
          type: "1f",
          value: 0,
        },
      }
    };
    
    // add the 2 buttons planes
    for(let key in this.options.buttons) {
      // offset time a bit for nextButton
      if(key === "nextButton") buttonParams.uniforms.time.value = 90;
      let buttonPlane = this.curtains.addPlane(this.options.buttons[key], buttonParams);
      
      if(buttonPlane) {
        // custom additional data
        buttonPlane.userData = {
          name: key,
          grow: 0,
          growTween: null,
        };
        
        // add event listeners when our plane is ready
        buttonPlane.onReady( () => {
          // mouse enter
          buttonPlane.htmlElement.addEventListener("mouseenter", () => {
            if (buttonPlane.userData.growTween) buttonPlane.userData.growTween.kill();

            buttonPlane.userData.growTween = TweenMax.to(buttonPlane.userData, 0.5, {
              grow: 1,
              ease: Power2.easeOut,
              onUpdate: () => {
                buttonPlane.uniforms.hoverEffect.value = buttonPlane.userData.grow;
              },
              onComplete: () => {
                buttonPlane.userData.growTween = null;
              }
            });
          });
          
          // mouse leave
          buttonPlane.htmlElement.addEventListener("mouseleave", () => {
            if (buttonPlane.userData.growTween) buttonPlane.userData.growTween.kill();

            buttonPlane.userData.growTween = TweenMax.to(buttonPlane.userData, 0.5, {
              grow: 0,
              ease: Power2.easeOut,
              onUpdate: () => {
                buttonPlane.uniforms.hoverEffect.value = buttonPlane.userData.grow;
              },
              onComplete: () => {
                buttonPlane.userData.growTween = null;
              }
            });
          });
        }).onRender( () => {
          buttonPlane.uniforms.time.value++;
        });
      }
    }
  }
}

window.addEventListener("load", () => {
  const slideshow = new WebGLSlideshow({
    webGLContainer: document.getElementById("curtains-canvas"),
    slideshow: document.getElementById("slides-list"),
    slides: document.getElementsByClassName("slide"),
    prevButton: document.getElementById("previous-slide"),
    nextButton: document.getElementById("next-slide"),
    duration: 1,
  });
});
  }
}
</script>
<style lang="scss">
body {
  margin: 0;
  background: #141414;
  font-family: 'Yeseva One', Arial;
  overflow: hidden;
}

#curtains-canvas {
  position: absolute;
  z-index: 0;
  height: 100vh;
  width: 100vw;
}

#slideshow {
  position: relative;
  z-index: 1;
  height: 100vh;
  width: 100vw;
}

.slideshow-button {
  position: absolute;
  z-index: 2;
  font-size: 2vw;
  top: calc(50% - 2.5em);
  width: 4.5em;
  height: 4.5em;
  box-sizing: border-box;
  padding: 0.5em;
  -webkit-appearance: none;
  border: 0;
  background: transparent;
  outline: 0;
  color: #eeeeee;
  font-family: inherit;
  cursor: pointer;
}

#previous-slide {
  left: 1.5em;
  transform: rotateZ(180deg);
}

#next-slide {
  right: 1.5em;
}

#slides-list {
  font-size: 2vw;
  position: absolute;
  top: 1.5em;
  right: 1.5em;
  bottom: 1.5em;
  left: 1.5em;
}

.slide {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  
  &-title {
    position: absolute;
    top: calc(50% - 0.7em);
    line-height: 1.4;
    right: 0;
    left: 0;
    will-change: transform;
    transform: translate3d(0, 0, 0) scale3d(1.15, 1.15, 1);
    text-align: center;
    pointer-events: none;
    color: #f21933;
    margin: 0;
    font-size: 3vw;
    font-weight: normal;

    opacity: 0;

    &:after {
      content: attr(data-title);
      position: absolute;
      z-index: -1;
      color: white;
      transform: translate3d(calc(-100% + 2px), 1px, 0);
    }
  }
  
  img {
    width: auto;
    height: auto;
    min-width: 100%;
    min-height: 100%;
    object-fit: contain;

    /* hide image */
    opacity: 0;
  }
}

@media screen and (max-width: 1000px) {
  #slides-list, .slideshow-button {
    font-size: 3vw;
  }
  
  .slide-title {
    font-size: 5vw;
  }
}
</style>
