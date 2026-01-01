# Scarlett-player
Scarlett-player
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Scarlett Player</title>
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />

  <style>
    :root{
      --bg:#0b0f19;--panel:#121a2a;--panel2:#0f1625;
      --text:#eef2ff;--muted:#aab3c5;
      --btn:#1d2a44;--btnHover:#253657;
      --radius:18px;--danger:#ff6b6b
    }
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial;background:var(--bg);color:var(--text)}
    .wrap{min-height:100%;display:grid;grid-template-rows:auto 1fr auto;padding:14px;gap:12px;max-width:1100px;margin:0 auto}
    header{display:flex;justify-content:space-between;align-items:center;padding:12px 14px;border-radius:var(--radius);
      background:linear-gradient(180deg,rgba(122,162,255,.18),rgba(122,162,255,.04));border:1px solid rgba(255,255,255,.08)}
    .title strong{font-size:16px}.title span{font-size:12px;color:var(--muted)}
    button{border:1px solid rgba(255,255,255,.1);background:var(--btn);color:var(--text);
      padding:10px 12px;border-radius:14px;font-weight:650;cursor:pointer}
    button:hover{background:var(--btnHover)}
    button.secondary{background:transparent;color:var(--muted)}
    button.danger{background:rgba(255,107,107,.12);border-color:rgba(255,107,107,.35)}
    main{display:grid;grid-template-columns:340px 1fr;gap:12px}
    .menu{background:var(--panel);border-radius:var(--radius);padding:12px;display:grid;gap:10px;align-content:start}
    .tile{padding:16px;border-radius:18px;background:rgba(255,255,255,.06);border:1px solid rgba(255,255,255,.1);text-align:left}
    .tile strong{font-size:16px}.tile span{font-size:12px;color:var(--muted);display:block;margin-top:6px}
    .player{background:var(--panel2);border-radius:var(--radius);overflow:hidden;display:grid;grid-template-rows:auto 1fr}
    .now{padding:12px;border-bottom:1px solid rgba(255,255,255,.08)}
    .frameWrap{position:relative;width:100%;height:100%;min-height:360px}
    #player{position:absolute;inset:0;width:100%;height:100%;background:#000}
    footer{font-size:12px;color:var(--muted);padding-bottom:8px}
    @media(max-width:900px){main{grid-template-columns:1fr}}
  </style>
</head>
<body>

<div class="wrap">
  <header>
    <div class="title">
      <strong>Scarlett Player</strong>
      <span>Buttons only · auto-restart for single videos</span>
    </div>
    <div>
      <button class="secondary" onclick="goHome()">Home</button>
      <button class="danger" onclick="stopPlayback()">Stop</button>
    </div>
  </header>

  <main>
    <section class="menu" id="buttons"></section>

    <section class="player">
      <div class="now">
        <strong id="nowTitle">Choose a category</strong><br>
        <span id="nowSub" style="color:var(--muted);font-size:12px">Buttons on the left</span>
      </div>
      <div class="frameWrap">
        <!-- YouTube API mounts the player into this div -->
        <div id="player"></div>
      </div>
    </section>
  </main>

  <footer>
    iPad setup: Add to Home Screen → Guided Access → triple-click side button
  </footer>
</div>

<script>
/**
 * CONTENT CONFIG
 * - Animals = playlist
 * - All others = single video with auto-restart on end
 */
const CONTENT = [
  { label:"Animals",  subtitle:"Animal videos", type:"playlist", id:"PLNxd9fYeqXea4N3q7_Au1eovEWdQj1KJK" },
  { label:"Kids Bop", subtitle:"Kids music",    type:"video",    id:"yzUKlvhlsyI" },
  { label:"Lullabies",subtitle:"Sleep & calm",  type:"video",    id:"Ddf6UKyxDwo" },
  { label:"Music",    subtitle:"Approved music",type:"video",    id:"d8S4VuWPYto" },
  { label:"ASMR",     subtitle:"Calm sensory",  type:"video",    id:"f63gAuLHnBo" }
];

const buttonsEl = document.getElementById("buttons");
const nowTitle = document.getElementById("nowTitle");
const nowSub = document.getElementById("nowSub");

let player = null;
let currentItem = null;

// Build the button UI
CONTENT.forEach(item => {
  const btn = document.createElement("button");
  btn.className = "tile";
  btn.innerHTML = `<strong>${item.label}</strong><span>${item.subtitle}</span>`;
  btn.onclick = () => loadItem(item);
  buttonsEl.appendChild(btn);
});

// Load the YouTube IFrame API
const tag = document.createElement('script');
tag.src = "https://www.youtube.com/iframe_api";
document.head.appendChild(tag);

// Called by the YouTube API when it's ready
window.onYouTubeIframeAPIReady = () => {
  player = new YT.Player('player', {
    width: '100%',
    height: '100%',
    videoId: '', // start empty
    playerVars: {
      autoplay: 1,
      playsinline: 1,
      rel: 0,
      modestbranding: 1,
      // These reduce UI but YouTube ultimately controls what is shown.
      controls: 1
    },
    events: {
      onStateChange: onPlayerStateChange
    }
  });
};

function loadItem(item){
  currentItem = item;
  nowTitle.textContent = item.label;
  nowSub.textContent = item.subtitle;

  if (!player || typeof player.loadVideoById !== "function") {
    // API not ready yet; retry shortly
    setTimeout(() => loadItem(item), 200);
    return;
  }

  if (item.type === "playlist") {
    // Start playlist (Animals)
    player.loadPlaylist({ listType: "playlist", list: item.id, index: 0, startSeconds: 0 });
  } else {
    // Single video
    player.loadVideoById({ videoId: item.id, startSeconds: 0 });
  }
}

// Auto-restart logic: when a single video ends, restart it
function onPlayerStateChange(event){
  if (!currentItem) return;

  // YT.PlayerState.ENDED === 0
  if (event.data === YT.PlayerState.ENDED) {
    if (currentItem.type === "video") {
      // restart from beginning
      player.seekTo(0, true);
      player.playVideo();
    }
    // For playlists, we do nothing (playlist continues naturally)
  }
}

function goHome(){
  nowTitle.textContent = "Choose a category";
  nowSub.textContent = "Buttons on the left";
}

function stopPlayback(){
  currentItem = null;
  nowTitle.textContent = "Stopped";
  nowSub.textContent = "Select a button to play";
  if (player && typeof player.stopVideo === "function") player.stopVideo();
}
</script>

</body>
</html>
