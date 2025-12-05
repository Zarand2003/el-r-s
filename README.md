# eleres
<!DOCTYPE html>
<html lang="hu">
<head>
  <meta charset="UTF-8">
  <title>Rajzolós felület</title>
  <style>
    body {
      margin: 0;
      background: black;
      color: white;
      font-family: sans-serif;
      text-align: center;
    }
    #instructions {
      margin: 10px;
      font-size: 1.2em;
    }
    #canvas-container {
      position: relative;
    }
    canvas {
      display: block;
      margin: 0 auto;
      background: black;
      touch-action: none;
      border: 1px solid white;
    }
    #buttons {
      margin: 10px;
    }
    button {
      margin: 0 5px;
      padding: 8px 16px;
      font-size: 1em;
    }
  </style>
</head>
<body>
  <div id="buttons">
    <button id="resetBtn">Újra</button>
    <button id="doneBtn">Kész</button>
  </div>
  <div id="instructions">jelölje be kényelmes tartomány területét</div>
  <div id="canvas-container">
    <canvas id="drawingCanvas" width="400" height="700"></canvas>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <script>
    // Supabase config
    const SUPABASE_URL = 'https://ahzhydrbmpybyosgvekb.supabase.co';
    const SUPABASE_KEY = 'sb_publishable_WQKWTmLYIxq3tUdfFm_6Dw_U6RAb0jy';
    const BUCKET_NAME = 'egykzestartomany';
    const supabase = Supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

    // Google Form
    const FORM_URL = 'https://docs.google.com/forms/u/0/d/e/1FAIpQLSfUxMmPoLPqHGZKovDs2eukDpj67zGGKkBYuD7xtBwdSgRE8Q/formResponse';
    const EASY_ENTRY = 'entry.2090430824';
    const HARD_ENTRY = 'entry.1822346389';

    // Canvas setup
    const canvas = document.getElementById('drawingCanvas');
    const ctx = canvas.getContext('2d');
    let drawing = false;

    function startDraw(e){
      drawing = true;
      draw(e);
    }

    function endDraw(e){
      drawing = false;
      ctx.beginPath();
    }

    function draw(e){
      if(!drawing) return;
      e.preventDefault();
      const rect = canvas.getBoundingClientRect();
      let x, y;
      if(e.touches){
        x = e.touches[0].clientX - rect.left;
        y = e.touches[0].clientY - rect.top;
      } else {
        x = e.clientX - rect.left;
        y = e.clientY - rect.top;
      }
      ctx.lineWidth = 20;
      ctx.lineCap = 'round';
      ctx.strokeStyle = 'white';
      ctx.lineTo(x, y);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(x, y);
    }

    canvas.addEventListener('mousedown', startDraw);
    canvas.addEventListener('mouseup', endDraw);
    canvas.addEventListener('mouseout', endDraw);
    canvas.addEventListener('mousemove', draw);

    canvas.addEventListener('touchstart', startDraw);
    canvas.addEventListener('touchend', endDraw);
    canvas.addEventListener('touchmove', draw);

    document.getElementById('resetBtn').addEventListener('click', () => {
      ctx.clearRect(0,0,canvas.width,canvas.height);
    });

    let step = 1; // 1 = kényelmes, 2 = kibővített

    document.getElementById('doneBtn').addEventListener('click', async () => {
      const dataUrl = canvas.toDataURL('image/png');
      const filename = `drawing_${Date.now()}.png`;

      // Upload to Supabase
      const { data, error } = await supabase
        .storage
        .from(BUCKET_NAME)
        .upload(filename, dataURLtoBlob(dataUrl));

      if(error){
        alert('Hiba a feltöltés során: ' + error.message);
        return;
      }

      // Get public URL
      const { publicUrl, error: urlError } = supabase
        .storage
        .from(BUCKET_NAME)
        .getPublicUrl(filename);

      if(urlError){
        alert('Hiba a public URL lekérésekor: ' + urlError.message);
        return;
      }

      if(step === 1){
        // Move to step 2
        step = 2;
        ctx.clearRect(0,0,canvas.width,canvas.height);
        document.getElementById('instructions').textContent = 'jelölje be kibővített tartomány területét';
        window.firstImageURL = publicUrl;
      } else {
        // Both steps done, redirect to Google Form
        const secondImageURL = publicUrl;
        const formUrl = FORM_URL + `?${EASY_ENTRY}=${encodeURIComponent(window.firstImageURL)}&${HARD_ENTRY}=${encodeURIComponent(secondImageURL)}`;
        window.location.href = formUrl;
      }
    });

    function dataURLtoBlob(dataurl) {
      const arr = dataurl.split(','), mime = arr[0].match(/:(.*?);/)[1],
      bstr = atob(arr[1]), n = bstr.length, u8arr = new Uint8Array(n);
      for (let i = 0; i < n; i++) u8arr[i] = bstr.charCodeAt(i);
      return new Blob([u8arr], {type:mime});
    }
  </script>
</body>
</html>
