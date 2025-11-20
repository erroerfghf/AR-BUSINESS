<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" />
  <title>Mini‑Onirix — WebAR Menu (MindAR + Three.js)</title>
  <style>
    html,body{height:100%;margin:0;font-family:Inter,system-ui,Segoe UI,Roboto,Arial}
    #ar-container{position:relative;width:100%;height:100vh;background:#000}
    canvas{width:100%;height:100%}
    .ui{position:absolute;left:12px;top:12px;display:flex;gap:8px;flex-direction:column;z-index:5}
    .btn{background:#111;color:#fff;padding:8px 12px;border-radius:10px;border:0;backdrop-filter: blur(6px)}
    .bottom-ui{position:absolute;left:50%;transform:translateX(-50%);bottom:16px;display:flex;gap:8px;z-index:5}
    .panel{background:rgba(0,0,0,0.45);padding:10px;border-radius:10px;color:#fff}
    .hint{font-size:13px;opacity:.9}
    input[type=file]{display:none}
  </style>
</head>
<body>
  <!--
    Mini-Onirix single-file WebAR demo
    - Uses MindAR (image tracking) + three.js to show a glb model on top of an image target
    - Supports: load model per ?model=filename.glb, pinch-to-zoom, 1-finger drag -> move on X/Z, 2-finger rotate, take photo, change model file locally
    - Hosting: put this file on any static host (GitHub Pages, Netlify, Vercel, your server)
    - For one QR = one model: generate QR pointing to this page with ?model=modelname.glb (e.g. https://your.site/ar.html?model=burger.glb)

    IMPORTANT: you must also create a MindAR image target file (.mind) for the printed menu image or target image.
    Use MindAR tools to convert your printed menu image into a target (mind file). Place it alongside this page and point mindarComponent to it (see `targets.mind` variable below).
  -->

  <div id="ar-container"></div>

  <div class="ui">
    <div class="panel">
      <div style="display:flex;gap:8px;align-items:center">
        <button id="btn-photo" class="btn">📸 Capture</button>
        <label class="btn" for="file-input">🖼️ Change Model</label>
        <input id="file-input" type="file" accept=".glb,.gltf" />
      </div>
      <div class="hint">Model: <span id="model-name">(default)</span></div>
    </div>
  </div>

  <div class="bottom-ui">
    <div class="panel">
      <div class="hint">Controls: 1-finger drag → move • pinch → scale • two-finger rotate → rotate</div>
      <div class="hint">Tip: Create a QR that opens this page with <code>?model=yourmodel.glb</code> to tie a QR to a single model.</div>
    </div>
  </div>

  <!-- libraries from CDNs -->
  <script src="https://cdn.jsdelivr.net/npm/three@0.158.0/build/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.158.0/examples/js/loaders/GLTFLoader.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.158.0/examples/js/controls/OrbitControls.js"></script>
  <!-- MindAR (image) bundle for three.js -->
  <script src="https://cdn.jsdelivr.net/npm/mind-ar@1.1.4/dist/mindar-image-three.prod.js"></script>

  <script>
    // ----------------- CONFIG -----------------
    // Name/path of the MindAR target file (generated from your menu image)
    const targets = {
      mind: './targets.mind', // <-- create and put your .mind file here (see instructions below)
      image: './target.jpg'   // optional: the original image for debugging
    };

    // default model file (if no ?model param)
    const defaultModel = './models/default.glb';

    // ------------------------------------------

    (async()=>{
      const urlParams = new URLSearchParams(window.location.search);
      const modelParam = urlParams.get('model');
      const modelToLoad = modelParam ? decodeURIComponent(modelParam) : defaultModel;
      document.getElementById('model-name').innerText = modelToLoad.split('/').pop();

      // MindAR + Three.js setup
      const mindarThree = new window.MINDAR.IMAGE.MindARThree({
        container: document.querySelector('#ar-container'),
        imageTargetSrc: targets.mind,
        uiLoading: "", uiScanning: "", uiError: "",
        maxTrack:1
      });

      const {renderer, scene, camera} = mindarThree;
      const light = new THREE.HemisphereLight(0xffffff, 0xbbbbff, 1);
      scene.add(light);
      const dir = new THREE.DirectionalLight(0xffffff, 0.5);
      dir.position.set(0,1,0);
      scene.add(dir);

      // anchor: model will attach to anchor group for the first target index
      const anchor = mindarThree.addAnchor(0);

      // parent object for easier transforms
      const modelGroup = new THREE.Group();
      modelGroup.position.set(0,0,0);
      anchor.group.add(modelGroup);

      // load GLB model function
      const loader = new THREE.GLTFLoader();
      let currentModel = null;
      async function loadModel(path){
        if(currentModel){ modelGroup.remove(currentModel); currentModel = null; }
        try{
          const gltf = await new Promise((res,rej)=>loader.load(path,res,undefined,rej));
          currentModel = gltf.scene;
          // set reasonable scale if needed
          currentModel.scale.set(0.15,0.15,0.15);
          // center on modelGroup
          modelGroup.add(currentModel);
        }catch(e){
          console.error('Failed to load model',e);
        }
      }

      // Initial model
      await loadModel(modelToLoad);

      // Interaction: touch-based translate/scale/rotate
      let ongoingTouches = [];
      let lastDistance = null;
      let lastAngle = null;

      function getTouches(evt){
        return Array.from(evt.touches || []);
      }

      function distance(p1,p2){
        const dx = p1.clientX-p2.clientX; const dy = p1.clientY-p2.clientY; return Math.hypot(dx,dy);
      }
      function angle(p1,p2){ return Math.atan2(p2.clientY-p1.clientY,p2.clientX-p1.clientX); }

      // Map 2D drag to small changes in X/Z on the modelGroup
      let lastTouch = null;
      renderer.domElement.addEventListener('touchstart', (e)=>{
        if(e.touches.length===1){ lastTouch = {x:e.touches[0].clientX,y:e.touches[0].clientY}; }
        if(e.touches.length===2){ lastDistance = distance(e.touches[0], e.touches[1]); lastAngle = angle(e.touches[0], e.touches[1]); }
      });

      renderer.domElement.addEventListener('touchmove', (e)=>{
        e.preventDefault();
        if(e.touches.length===1 && lastTouch){
          const touch = e.touches[0];
          const dx = (touch.clientX - lastTouch.x) / window.innerWidth; // normalized
          const dy = (touch.clientY - lastTouch.y) / window.innerHeight;
          // move on X and Z
          modelGroup.position.x += dx * 1.2; // sensitivity tune
          modelGroup.position.z += dy * 1.2;
          lastTouch = {x:touch.clientX,y:touch.clientY};
        } else if(e.touches.length===2){
          const d = distance(e.touches[0], e.touches[1]);
          const ang = angle(e.touches[0], e.touches[1]);
          if(lastDistance){
            const scaleChange = d / lastDistance;
            modelGroup.scale.multiplyScalar(scaleChange);
          }
          if(lastAngle){
            const angDiff = ang - lastAngle;
            modelGroup.rotation.y += angDiff;
          }
          lastDistance = d; lastAngle = ang;
        }
      }, {passive:false});

      renderer.domElement.addEventListener('touchend', (e)=>{
        if(e.touches.length===0){ lastTouch = null; lastDistance = null; lastAngle = null; }
      });

      // Capture photo
      document.getElementById('btn-photo').addEventListener('click', ()=>{
        // render once to ensure frame
        renderer.render(scene, camera);
        const dataUrl = renderer.domElement.toDataURL('image/jpeg', 0.9);
        const a = document.createElement('a'); a.href = dataUrl; a.download = 'ar_capture.jpg'; a.click();
      });

      // Local model change via file input
      document.getElementById('file-input').addEventListener('change', async (ev)=>{
        const file = ev.target.files[0];
        if(!file) return;
        const url = URL.createObjectURL(file);
        document.getElementById('model-name').innerText = file.name;
        await loadModel(url);
      });

      // Start AR
      await mindarThree.start();

      // render loop
      renderer.setAnimationLoop(()=>{ renderer.render(scene, camera); });

      // Helpful: pause/resume on visibility
      document.addEventListener('visibilitychange', ()=>{ if(document.hidden) mindarThree.pause(); else mindarThree.resume(); });

      // Expose some debug helpers on window
      window.modelGroup = modelGroup;
      window.loadModel = loadModel;

    })();
  </script>

  <!--
    HOW TO USE / DEPLOY
    1) Prepare MindAR target file (.mind):
       - Use MindAR's `mindar-image-target` tool to convert your printed menu image (the image your camera will scan) into a .mind file.
       - Put that .mind file next to this HTML as `targets.mind` or change `targets.mind` path above.
    2) Put your GLB models in /models/ (or any path) and set query param: https://your.site/ar.html?model=models/burger.glb
       - If you want one QR per model, generate a QR that points to that URL (with model param).
    3) Host the files on any static host (GitHub Pages, Netlify, Vercel, S3 + CloudFront, your VPS).
    4) Print the menu image (the same image used to create the .mind target) near the table, or embed it in the physical menu.
    5) Scan the QR printed on the menu which links to your hosted ar.html (with ?model=...) and the AR experience will open.

    NOTES & LIMITATIONS
    - This demo uses image tracking (user points camera at the printed target). Markerless plane detection (surface AR without a printed target) is more complex and often needs ARCore/ARKit or 8thWall.
    - Tweak scale/position defaults in loadModel() depending on your models' native scale.
    - If models are large, consider compressing or Draco-compressed glb.
  -->
</body>
</html>
