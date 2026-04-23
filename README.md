<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>Fast Galery</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">

  <!-- Firebase -->
  <script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-storage-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore-compat.js"></script>

  <style>
    body {
      margin: 0;
      font-family: 'Segoe UI', Arial;
      background: linear-gradient(135deg, #0f2027, #203a43, #2c5364);
      color: white;
    }

    h2 {
      padding: 20px;
      margin: 0;
      text-align: center;
      letter-spacing: 2px;
      background: rgba(0,0,0,0.3);
      backdrop-filter: blur(10px);
    }

    .search-box { padding: 15px 20px; }

    .search-box input {
      width: 100%;
      padding: 12px;
      border-radius: 25px;
      border: none;
      outline: none;
    }

    #list {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 12px;
      padding: 12px;
    }

    .item { position: relative; }

    .item img {
      width: 100%;
      height: 120px;
      object-fit: cover;
      border-radius: 12px;
      cursor: pointer;
    }

    .label {
      font-size: 12px;
      text-align: center;
      margin-top: 5px;
    }

    .delete-btn {
      position: absolute;
      top: 5px;
      right: 5px;
      background: red;
      border: none;
      color: white;
      border-radius: 50%;
      width: 25px;
      height: 25px;
      cursor: pointer;
    }

    .fab {
      position: fixed;
      bottom: 20px;
      right: 20px;
      width: 65px;
      height: 65px;
      background: #1e88e5;
      border-radius: 50%;
      color: white;
      font-size: 32px;
      border: none;
      cursor: pointer;
    }

    #upload { display: none; }

    #viewer {
      display: none;
      position: fixed;
      width: 100%;
      height: 100%;
      background: rgba(0,0,0,0.95);
      justify-content: center;
      align-items: center;
    }

    #viewer img {
      max-width: 90%;
      max-height: 80%;
      border-radius: 10px;
      touch-action: none;
    }

    #closeViewer {
      position: absolute;
      top: 20px;
      right: 25px;
      font-size: 35px;
      cursor: pointer;
    }

    .footer {
      text-align: center;
      font-size: 12px;
      padding: 10px;
    }
  </style>
</head>
<body>

<h2>✨ Fast Galery Cloud</h2>

<div class="search-box">
  <input type="text" id="search" placeholder="🔍 Cari foto..." onkeyup="cariFoto()">
</div>

<div id="list"></div>

<div id="viewer" onclick="tutupViewer()">
  <span id="closeViewer">×</span>
  <img id="viewerImg">
</div>

<input type="file" id="upload" accept="image/*">
<button class="fab" onclick="upload.click()">+</button>

<div class="footer">© 2026 Fast Galery</div>

<script>
// ================== FIREBASE ==================
const firebaseConfig = {
  apiKey: "ISI_API_KEY",
  authDomain: "ISI_PROJECT.firebaseapp.com",
  projectId: "ISI_PROJECT",
  storageBucket: "ISI_PROJECT.appspot.com",
  appId: "ISI_APP_ID"
};

firebase.initializeApp(firebaseConfig);
const storage = firebase.storage();
const dbCloud = firebase.firestore();

// ================== ELEMENT ==================
const list = document.getElementById("list");
const upload = document.getElementById("upload");
const viewerImg = document.getElementById("viewerImg");

let db;

// ================== INDEXEDDB ==================
const request = indexedDB.open("FastGalleryDB", 1);

request.onupgradeneeded = function(e) {
  db = e.target.result;
  db.createObjectStore("photos", { keyPath: "id", autoIncrement: true });
};

request.onsuccess = function(e) {
  db = e.target.result;
  loadData();
  loadCloud();
};

// ================== LOAD LOCAL ==================
function loadData() {
  const tx = db.transaction("photos", "readonly");
  const store = tx.objectStore("photos");
  const req = store.getAll();

  req.onsuccess = function() {
    req.result.forEach(item => {
      const url = URL.createObjectURL(item.file);
      tambahItem(url, item.nama, item.id);
    });
  };
}

// ================== LOAD CLOUD ==================
async function loadCloud() {
  const snapshot = await dbCloud.collection("photos")
    .orderBy("waktu", "desc")
    .get();

  snapshot.forEach(doc => {
    const data = doc.data();
    tambahItem(data.url, data.nama);
  });
}

// ================== SAVE LOCAL ==================
function simpanDB(file, nama) {
  const tx = db.transaction("photos", "readwrite");
  const store = tx.objectStore("photos");
  store.add({ file: file, nama: nama });
}

// ================== DELETE ==================
function hapusDB(id) {
  const tx = db.transaction("photos", "readwrite");
  const store = tx.objectStore("photos");
  store.delete(Number(id));
}

// ================== UI ==================
function tambahItem(src, nama, id = null) {
  const div = document.createElement("div");
  div.className = "item";
  div.setAttribute("data-name", nama.toLowerCase());
  if (id !== null) div.setAttribute("data-id", id);

  div.innerHTML = `
    <img src="${src}" onclick="bukaViewer(this.src)">
    <button class="delete-btn" onclick="hapusItem(this)">×</button>
    <div class="label">${nama}</div>
  `;
  list.appendChild(div);
}

// ================== UPLOAD ==================
upload.addEventListener("change", async function() {
  const file = this.files[0];
  if (!file) return;

  const nama = prompt("Nama foto:");
  if (!nama) return;

  const localURL = URL.createObjectURL(file);

  tambahItem(localURL, nama);
  simpanDB(file, nama);

  // CLOUD
  const ref = storage.ref("photos/" + Date.now() + "_" + file.name);
  await ref.put(file);

  const url = await ref.getDownloadURL();

  await dbCloud.collection("photos").add({
    nama: nama,
    url: url,
    waktu: Date.now()
  });

  this.value = "";
});

// ================== DELETE ==================
function hapusItem(btn) {
  if (confirm("Hapus foto?")) {
    const parent = btn.parentElement;
    const id = parent.getAttribute("data-id");

    if (id) hapusDB(id);
    parent.remove();
  }
}

// ================== SEARCH ==================
function cariFoto() {
  const keyword = document.getElementById("search").value.toLowerCase();
  document.querySelectorAll(".item").forEach(item => {
    const nama = item.getAttribute("data-name");
    item.style.display = nama.includes(keyword) ? "block" : "none";
  });
}

// ================== VIEWER ==================
function bukaViewer(src) {
  document.getElementById("viewer").style.display = "flex";
  viewerImg.src = src;
  resetZoom();
}

function tutupViewer() {
  document.getElementById("viewer").style.display = "none";
}

// ================== ZOOM + PAN ==================
let scale = 1, posX = 0, posY = 0;
let startX, startY, isDragging = false;

function updateTransform() {
  viewerImg.style.transform =
    `translate(${posX}px, ${posY}px) scale(${scale})`;
}

function resetZoom() {
  scale = 1;
  posX = 0;
  posY = 0;
  updateTransform();
}

viewerImg.addEventListener("wheel", e => {
  e.preventDefault();
  scale += e.deltaY * -0.001;
  scale = Math.min(Math.max(1, scale), 5);
  updateTransform();
});

viewerImg.addEventListener("mousedown", e => {
  if (scale <= 1) return;
  isDragging = true;
  startX = e.clientX - posX;
  startY = e.clientY - posY;
});

window.addEventListener("mousemove", e => {
  if (!isDragging) return;
  posX = e.clientX - startX;
  posY = e.clientY - startY;
  updateTransform();
});

window.addEventListener("mouseup", () => isDragging = false);

// TOUCH (PINCH + PAN)
let initialDistance = null;
let isPanning = false;

viewerImg.addEventListener("touchstart", e => {
  if (e.touches.length === 2) {
    initialDistance = getDistance(e.touches);
  } else if (e.touches.length === 1 && scale > 1) {
    isPanning = true;
    startX = e.touches[0].clientX - posX;
    startY = e.touches[0].clientY - posY;
  }
});

viewerImg.addEventListener("touchmove", e => {
  if (e.touches.length === 2) {
    const newDistance = getDistance(e.touches);
    scale *= newDistance / initialDistance;
    scale = Math.min(Math.max(1, scale), 5);
    initialDistance = newDistance;
    updateTransform();
  } else if (e.touches.length === 1 && isPanning) {
    posX = e.touches[0].clientX - startX;
    posY = e.touches[0].clientY - startY;
    updateTransform();
  }
});

viewerImg.addEventListener("touchend", () => {
  isPanning = false;
  initialDistance = null;
});

function getDistance(touches) {
  return Math.hypot(
    touches[0].clientX - touches[1].clientX,
    touches[0].clientY - touches[1].clientY
  );
}
</script>

</body>
</html>
