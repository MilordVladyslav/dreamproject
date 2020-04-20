<template>
  <div>
    <client-only>
    <div id="vertexShader">
      varying vec2 v_uv;
      varying vec2 v_uv1;
      uniform vec2 uvRate1;

      void main() {	
        v_uv = uv;
        vec2 _uv = uv - 0.5;
        v_uv1 = _uv;
        v_uv1*=uvRate1;
        v_uv1+=0.5;

        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
      }
    </div> 
    <div id="fragmentShader">
      uniform sampler2D u_tex;
      uniform sampler2D u_tex2;
      uniform sampler2D u_text;
      uniform sampler2D u_text2;
      uniform sampler2D videotex;
      uniform vec2 u_resolution;

      uniform float progress;
      uniform vec2 accel;
      uniform float u_time;

      varying vec2 v_uv;
      varying vec2 v_uv1;

      /*
      ----- Image Mirroring function
      */
      vec2 mirrored(vec2 v) {
          vec2 m = mod(v,2.);
          <!-- return mix(m,2.0-m,step(1.0,m)); -->
          return m;
      }

      /*
      Corner black to white and back to black, p - will be time;
      very fucking usefull thing for masking some effect for specific timeframe

      */
      float tri(float p) {
          return mix(p,1.0 - p, step(0.5,p))*2.;
      }

      void main (void) {

      vec2 uv = gl_FragCoord.xy/u_resolution.xy;
      float p = fract(progress);
      float delayValue = p*7. - v_uv.y*2.+ v_uv.x-1.; // our wave, -1 at the end to hide it from screen
      delayValue = clamp(delayValue, 0.,1.);

      vec2 translatevalue = p + delayValue*accel;
      vec2 translatevalue1 = vec2(-0.5,1.)*translatevalue;
      vec2 translatevalue2 = vec2(-0.5,1.)*(translatevalue - 1. -accel);

      vec2 translatevaluetext1 = vec2(-0.1,1.)*translatevalue;
      vec2 translatevaluetext2 = vec2(-0.1,1.)*(translatevalue - 1. -accel);

      vec2 w = sin(sin(u_time)*vec2(0,0.3) + v_uv.yx*vec2(0,4.))*vec2(0,0.5); 
      vec2 wtext = sin(sin(u_time)*vec2(0,0.2) + v_uv.yx*vec2(0,3.))*vec2(0,0.4); 
      // Curvature function for the curtain,
      // first part is sin tied to coordinates multiplied by vector to make it less strong
      vec2 xy = w*(tri(p)*0.5 + tri(delayValue)*0.5); // this function limits effect to the tree function
      vec2 xytext = wtext*(tri(p)*0.2 + tri(delayValue)*0.2);

      vec2 uv_distorted1 = v_uv1 + translatevalue1 + xy;
      vec2 uv_distorted2 = v_uv1 + translatevalue2 + xy;

      vec4 rgba1 = texture2D(u_tex,mirrored(uv_distorted1));
      vec4 rgba2 = texture2D(u_tex2,mirrored(uv_distorted2));

      vec4 rgba = mix(rgba1,rgba2,delayValue);

      // text:

      vec2 uv_distortedtext1 = v_uv1 + translatevaluetext1 + xytext;
      vec2 uv_distortedtext2 = v_uv1 + translatevaluetext2 + xytext;

      vec4 text1 = texture2D(u_text,mirrored(uv_distortedtext1));
      vec4 text2 = texture2D(u_text2,mirrored(uv_distortedtext2));

      // text rgb shift 

      text1.r = texture2D(u_text,mirrored(uv_distortedtext1) + vec2(0.0,0.01)*translatevaluetext1).r;
      text2.r = texture2D(u_text2,mirrored(uv_distortedtext2) + vec2(0.0,0.01)*translatevaluetext2).r;

      text1.g = texture2D(u_text,mirrored(uv_distortedtext1) + vec2(0.0,0.02)*translatevaluetext1).g;
      text2.g = texture2D(u_text2,mirrored(uv_distortedtext2) + vec2(0.0,0.02)*translatevaluetext2).g;

      text1.b = texture2D(u_text,mirrored(uv_distortedtext1) + vec2(0.0,0.03)*translatevaluetext1).b;
      text2.b = texture2D(u_text2,mirrored(uv_distortedtext2) + vec2(0.0,0.03)*translatevaluetext2).b;


      vec4 textcombined = mix(text1,text2,delayValue);

      vec4 textcolor = textcombined*vec4(1.0,0.0,0.0,1.0);
      //vec4 textcolor = textcombined;

      gl_FragColor = rgba+textcolor;

      //gl_FragColor = vec4(tri(p));
      //gl_FragColor = texture2D(videotex,v_uv);
      }
    </div>   

    <div class="vid-cont"> 
      <img id="video_mech" class="video" src="../assets/stardust_big.png"/>
      <img id="video_techie" class="video" src="../assets/squares_dark_big.png"/>
      <img id="video_third" class="video" src="../assets/math_squares_dark_biggest.png"/>
      <img id="video_fourth" class="video" src="../assets/square_across_dark_pattern_big.png">
      <img id="video_fifth" class="video" src="../assets/cube_triangle_pattern_big.png">
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
        }
      ]
    }

  },
  data() {
    return {
      currentPage: ''
    }
  },
  updated() {

    document.body.addEventListener('click', () => {
      alert(this.currentPage)
    })

    const scene = new THREE.Scene();

    const camera = new THREE.PerspectiveCamera(
        70,
        window.innerWidth / window.innerHeight,
        0.001, 100
      );
      camera.position.set( 0, 0, 1 );

    const renderer = new THREE.WebGLRenderer();
    renderer.setSize( window.innerWidth, window.innerHeight );
    document.body.appendChild( renderer.domElement );

    const clock = new THREE.Clock();

    // this function find mp4 from html by id and initializing Three.js video texture function
    const videoInit = function (name) {
      let v = document.getElementById(name); // getting video by name from html
      console.log(v.src)
      let videoOut = new THREE.TextureLoader().load(v.src); // initializing three.js video texture constructor
      videoOut.wrapS = THREE.RepeatWrapping;
      videoOut.wrapT = THREE.RepeatWrapping;
      videoOut.repeat.set( 1, 1 );
      // v.pause(); // start playing videos
      return videoOut; // output - actually what is passed to array
    }

    // Gallery array
    let gallery = [];

    // this method is pushing previously initialized videos in videoInit() function
    gallery.push(
        {
          id: 1,
          background: videoInit('video_mech'),
          text: new THREE.TextureLoader().load('https://raw.githubusercontent.com/MilordVladyslav/dreamproject/master/assets/portfolio_text.png'),
          route: '/portfolio'
        },
        {
          id: 2,
          background: videoInit('video_third'),
          text: new THREE.TextureLoader().load('https://raw.githubusercontent.com/MilordVladyslav/dreamproject/master/assets/blog_text.png'),
          route: '/blog'
        },
        {
          id: 3,
          background: videoInit('video_fourth'),
          text: new THREE.TextureLoader().load('https://raw.githubusercontent.com/MilordVladyslav/dreamproject/master/assets/contacts_text.png'),
          route: '/contacts'
        },


        // {
        //   background: videoInit('video_fifth'),
        //   text: new THREE.TextureLoader().load('https://raw.githubusercontent.com/MilordVladyslav/dreamproject/master/assets/portfolio.png'),
        //   route: '/contacts'

        // },
        // {
        //   background: videoInit('video_techie'),
        //   text: new THREE.TextureLoader().load('https://www.ivashnev.com/wp-content/uploads/2020/01/text-test.jpg')
        // }
      );
      gallery.sort(function(a, b) {
        return a.id - b.id;
      });




    const uniforms = {
      u_color: { value: new THREE.Color(0xffdd33) },
      u_tex:{},
      u_tex2:{},
      u_text:{value:new THREE.TextureLoader().load('https://raw.githubusercontent.com/MilordVladyslav/dreamproject/master/assets/portfolio_text.png')},
      u_text2: {value:new THREE.TextureLoader().load('https://raw.githubusercontent.com/MilordVladyslav/dreamproject/master/assets/blog_text.png')},
      u_time: { value: 0.0 },
      progress: {type: 'f', value: 0},
      u_mouse: { value:{ x:0.0, y:0.0 }},
      uvRate1:{value:new THREE.Vector2(1,1)},
      accel:{value:new THREE.Vector2(0.5,2)},
      u_resolution: { value:{ x:0, y:0 }}
    }





    const geometry = new THREE.PlaneGeometry( 1, 1, 1, 1 );

    const material = new THREE.ShaderMaterial( {
      vertexShader: document.getElementById( 'vertexShader' ).textContent,
      fragmentShader: document.getElementById('fragmentShader').textContent,
      uniforms: uniforms,      
    } );

    const plane = new THREE.Mesh( geometry, material );
    scene.add( plane );



    if ('ontouchstart' in window){
      document.addEventListener('touchmove', move);
    }else{
      window.addEventListener( 'resize', resize, false );
      document.addEventListener('mousemove', move);
    }

    function move(evt){
      uniforms.u_mouse.value.x = (evt.touches) ? evt.touches[0].clientX : evt.clientX;
      uniforms.u_mouse.value.y = (evt.touches) ? evt.touches[0].clientY : evt.clientY;
    }

    animate();

    // YA size

    function resize(){
      let w = window.innerWidth;
      let h = window.innerHeight;
      renderer.setSize(w,h);
      camera.aspect=w/h;
      // if(window.innerWidth <= 800  &&  window.innerHeight <= 600 ) {
      //   material.uniforms.uvRate1.value.y = (h/1)/(w/0.58);
      // } else {
        material.uniforms.uvRate1.value.y = (h+h/2)/(w/1);
      // }
      // material.uniforms.uvRate1.value.y = (h/1)/(w/0.78); //custom aspect ratio ??
    //calculate scene
      let dist = camera.position.z-plane.position.z;
      let height = 1;
      camera.fov=2*(180/Math.PI)*Math.atan(height/(2*dist));
      //if(w/h>1) {
        //plane.scale.x=plane.scale.y=1.0*w/h;
        plane.scale.x=w/h;
    // }
      camera.updateProjectionMatrix();
    }

    resize();

    function animate() {
      requestAnimationFrame( animate );
      uniforms.u_time.value = clock.getElapsedTime();
      renderer.render( scene, camera );
    }

    // Magical Scroll function, this is the most important part of this slider effect

    let speed =0;
    let position = 0;
    let lastY
    let average;
    $(document).bind('touchmove', function (e){
        var currentY = e.originalEvent.touches[0].clientY;
        if(currentY > lastY){
          average = lastY + currentY
        }else if(currentY < lastY){
          average = lastY - currentY
          speed += -1 * 0.02
        }
        lastY = currentY;
    });

    document.addEventListener('wheel',function(event) {                       
        speed += event.deltaY*0.0006; // vertical scroll speed                    
    })

    //let tll = new TimelineMax();

    const raf = () => {
      position += speed;
      speed *=0.7; // we slowing down our scroll by multiplying
      
      let i = Math.round(position) 
      let dif = i - position; // distance between yellow point and white point
      position += dif*0.02 // normalizing position
      // 
      if(Math.abs(i-position)<0.001) {
        position=i;
      }
    material.uniforms.progress.value = position;
    let curslide =  ((Math.floor(position) + 0)%gallery.length + gallery.length)%gallery.length;  
    let nextslide = (((Math.floor(position) + 0)%gallery.length +1) + gallery.length)%gallery.length;
    let curposition =  ((position + 0)%gallery.length + gallery.length)%gallery.length;
    uniforms.u_tex.value = gallery[curslide].background; 
    uniforms.u_tex2.value = gallery[nextslide].background;
    uniforms.u_text.value = gallery[curslide].text;
    uniforms.u_text2.value = gallery[nextslide].text
    const navigate = Math.round(curposition) !== gallery.length ? Math.round(curposition) : 0 
    console.log(curposition)
    this.currentPage = gallery[Math.round(navigate)].route
    
    if (curposition == curslide) {
      
        function Q(root, selector) {
        if (typeof root === "string") {
          selector = root
          root = document
        }
        return root.querySelectorAll(selector)
      }
    }
    window.requestAnimationFrame(raf);
    }
    raf();


  }
}
</script>
<style lang="scss">
  #vertexShader, #fragmentShader {
    display: none
  }
  body { margin: 0;padding:0 }
  canvas { 
    width: 100%; 
    height: 100%;
    display:block;
  }
  .vid-cont {position:absolute;top:50px;right:50px;overflow:hidden}
  .video {display:none;border:1px solid rgba(255, 255, 255, 0.1);width:1px;margin-bottom:10px;background-color:red}
  .current {border:1px solid rgba(255, 255, 255, 0.5)}
  #video_techie {right:15vw;}
  #video_problem {right:30vw}
  #video_math {right:45vw}
  #video_scientist {right:60vw}
  #info {
    position: fixed;
    z-index: 2;
    width: 110px;
    height: 40px;
    color: white;
    border: 3px dashed silver;

  }
</style>
