```text

Prompt:
===========

i need to create a simple html, css, javascript and python based html catalogue which can be browsed offline
the catalogue should render first page as Hero video (100% view port) and option to navigate to categories
Categories html page should be of following structure
Left frame - list all first level folders under Product_catalogue folder on left frame
- on clicking a product category in left frame it should display first product and product's medata data in right frame

Right frame - list product image full view of image
Along with product image, it should display product metadata picked from product.json data file
There should be option to navigate next & previous product view under selected product category

towards bottom of right frame there should be frame (equal to right frame width) which can display an image bar




Solution:
==========

# Offline Product Catalogue

This approach uses **HTML, CSS, JavaScript, and Python only**—no internet connection, database, or external libraries required.

- The first page is a **full-viewport hero video**
- The catalogue page shows:
  - **Left panel:** first-level folders inside `Product_catalogue`
  - **Right panel:** selected product image and metadata from `product.json`
  - **Previous / Next** controls for products in the selected category
  - A **bottom image bar** containing product thumbnails

Run it through Python locally, then browse it at `http://127.0.0.1:8000`.

---

## 1. Folder Structure

Create the project like this:

```text
offline_catalogue/
│
├── server.py
├── Product_catalogue/
│   ├── Furniture/
│   │   ├── Chair_001/
│   │   │   ├── product.json
│   │   │   ├── main.jpg
│   │   │   └── detail.jpg
│   │   │
│   │   └── Table_001/
│   │       ├── product.json
│   │       └── main.jpg
│   │
│   └── Lighting/
│       └── Lamp_001/
│           ├── product.json
│           └── main.jpg
│
└── site/
    ├── index.html
    ├── catalogue.html
    ├── styles.css
    ├── app.js
    └── assets/
        └── hero-video.mp4
```

> Each **first-level folder** inside `Product_catalogue` is treated as a product **category**.

---

## 2. Product Metadata File: `product.json`

Put a `product.json` file inside every product folder.

Example: `Product_catalogue/Furniture/Chair_001/product.json`

```json
{
  "name": "Nordic Lounge Chair",
  "images": [
    "main.jpg",
    "detail.jpg"
  ],
  "metadata": {
    "Product Code": "CHR-001",
    "Material": "Oak wood, fabric upholstery",
    "Colour": "Natural oak / Grey",
    "Dimensions": "720 × 760 × 800 mm",
    "Designer": "Studio North",
    "Description": "A comfortable lounge chair with a minimalist Scandinavian silhouette."
  }
}
```

- `name` is optional. If omitted, the product folder name is used.
- `images` is optional. If omitted, Python automatically finds image files in the product folder.
- Supported images: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`

---

## 3. Python Server: `server.py`

This server:

- Serves the HTML catalogue
- Reads folders inside `Product_catalogue`
- Reads each `product.json`
- Makes catalogue data available to JavaScript at `/api/catalogue`
- Serves product images safely from `/media/...`

```python
from http.server import HTTPServer, SimpleHTTPRequestHandler
from pathlib import Path
from urllib.parse import unquote, urlparse, quote
import json
import mimetypes

BASE_DIR = Path(__file__).resolve().parent
SITE_DIR = BASE_DIR / "site"
CATALOGUE_DIR = BASE_DIR / "Product_catalogue"

IMAGE_EXTENSIONS = {".jpg", ".jpeg", ".png", ".webp", ".gif"}


def safe_path(root: Path, relative_path: str):
    """Return a safe file path inside root, or None if traversal is attempted."""
    root = root.resolve()
    target = (root / relative_path).resolve()

    if target == root or root in target.parents:
        return target

    return None


def media_url(file_path: Path):
    relative_path = file_path.relative_to(CATALOGUE_DIR).as_posix()
    return "/media/" + quote(relative_path)


def load_product(product_folder: Path):
    metadata_file = product_folder / "product.json"
    product_data = {}

    if metadata_file.exists():
        try:
            product_data = json.loads(metadata_file.read_text(encoding="utf-8"))
        except json.JSONDecodeError:
            print(f"Warning: Invalid JSON in {metadata_file}")

    configured_images = product_data.get("images", [])
    images = []

    if isinstance(configured_images, list):
        for image_name in configured_images:
            image_path = product_folder / image_name
            if image_path.exists() and image_path.suffix.lower() in IMAGE_EXTENSIONS:
                images.append(media_url(image_path))

    if not images:
        images = [
            media_url(image_path)
            for image_path in sorted(product_folder.iterdir())
            if image_path.is_file()
            and image_path.suffix.lower() in IMAGE_EXTENSIONS
        ]

    metadata = product_data.get("metadata", {})

    if not isinstance(metadata, dict):
        metadata = {}

    # Add top-level fields other than name, images, and metadata.
    for key, value in product_data.items():
        if key not in {"name", "images", "metadata"}:
            metadata[key] = value

    return {
        "id": product_folder.name,
        "name": product_data.get("name", product_folder.name.replace("_", " ")),
        "images": images,
        "metadata": metadata,
    }


def build_catalogue():
    categories = []

    if not CATALOGUE_DIR.exists():
        return {"categories": categories}

    for category_folder in sorted(CATALOGUE_DIR.iterdir()):
        if not category_folder.is_dir() or category_folder.name.startswith("."):
            continue

        products = []

        for product_folder in sorted(category_folder.iterdir()):
            if not product_folder.is_dir() or product_folder.name.startswith("."):
                continue

            product = load_product(product_folder)

            # Include folders even when an image is missing,
            # but they will show an empty image area.
            products.append(product)

        categories.append({
            "id": category_folder.name,
            "name": category_folder.name.replace("_", " "),
            "products": products
        })

    return {"categories": categories}


class CatalogueHandler(SimpleHTTPRequestHandler):
    def do_GET(self):
        parsed_url = urlparse(self.path)
        path = unquote(parsed_url.path)

        if path == "/api/catalogue":
            catalogue = build_catalogue()
            response = json.dumps(catalogue, ensure_ascii=False).encode("utf-8")

            self.send_response(200)
            self.send_header("Content-Type", "application/json; charset=utf-8")
            self.send_header("Content-Length", str(len(response)))
            self.end_headers()
            self.wfile.write(response)
            return

        if path.startswith("/media/"):
            relative_path = path.removeprefix("/media/")
            file_path = safe_path(CATALOGUE_DIR, relative_path)

            if not file_path or not file_path.is_file():
                self.send_error(404, "Media file not found")
                return

            self.send_file(file_path)
            return

        if path == "/":
            path = "/index.html"

        file_path = safe_path(SITE_DIR, path.lstrip("/"))

        if not file_path or not file_path.is_file():
            self.send_error(404, "File not found")
            return

        self.send_file(file_path)

    def send_file(self, file_path: Path):
        content_type, _ = mimetypes.guess_type(str(file_path))
        content_type = content_type or "application/octet-stream"

        try:
            file_data = file_path.read_bytes()

            self.send_response(200)
            self.send_header("Content-Type", content_type)
            self.send_header("Content-Length", str(len(file_data)))
            self.end_headers()
            self.wfile.write(file_data)
        except OSError:
            self.send_error(500, "Could not read file")

    def log_message(self, format, *args):
        print(f"[Catalogue] {args[0]}")


if __name__ == "__main__":
    server_address = ("127.0.0.1", 8000)

    print("Offline catalogue running at:")
    print("http://127.0.0.1:8000")

    print("\nPress Ctrl+C to stop the server.")

    HTTPServer(server_address, CatalogueHandler).serve_forever()
```

---

## 4. Hero Page: `site/index.html`

The hero video fills the complete browser viewport.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Product Catalogue</title>
  <link rel="stylesheet" href="styles.css">
</head>

<body class="hero-page">
  <main class="hero">
    <video class="hero-video" autoplay muted loop playsinline poster="assets/hero-poster.jpg">
      <source src="assets/hero-video.mp4" type="video/mp4">
      Your browser does not support HTML video.
    </video>

    <div class="hero-overlay"></div>

    <section class="hero-content">
      <p class="hero-kicker">Product Collection</p>
      <h1>Explore Our Catalogue</h1>
      <p class="hero-description">
        Browse products, specifications, materials, and imagery.
      </p>

      <a class="hero-button" href="catalogue.html">
        Browse Categories
      </a>
    </section>
  </main>
</body>
</html>
```

Place your video at:

```text
site/assets/hero-video.mp4
```

---

## 5. Catalogue Page: `site/catalogue.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Product Catalogue</title>
  <link rel="stylesheet" href="styles.css">
</head>

<body class="catalogue-page">
  <header class="catalogue-header">
    <a href="index.html" class="brand">PRODUCT CATALOGUE</a>
    <span id="categoryTitle" class="current-category">Loading catalogue…</span>
  </header>

  <main class="catalogue-layout">
    <aside class="category-panel">
      <p class="panel-label">Categories</p>
      <nav id="categoryList" class="category-list" aria-label="Product categories"></nav>
    </aside>

    <section class="product-panel">
      <div id="emptyState" class="empty-state" hidden>
        <h2>No products found</h2>
        <p>
          Add category folders and product folders inside
          <code>Product_catalogue</code>.
        </p>
      </div>

      <div id="productContent" class="product-content">
        <div class="product-main">
          <div class="image-area">
            <img id="productImage" src="" alt="">
            <div id="noImageMessage" class="no-image-message" hidden>
              No product image available
            </div>
          </div>

          <div class="product-information">
            <p id="productPosition" class="product-position"></p>
            <h1 id="productName">Product Name</h1>

            <dl id="metadataList" class="metadata-list"></dl>

            <div class="product-navigation">
              <button id="previousButton" class="nav-button" type="button">
                ← Previous
              </button>

              <button id="nextButton" class="nav-button" type="button">
                Next →
              </button>
            </div>
          </div>
        </div>

        <section class="thumbnail-panel">
          <div class="thumbnail-header">
            <p class="panel-label">Products in Category</p>
            <span id="thumbnailCount"></span>
          </div>

          <div id="thumbnailBar" class="thumbnail-bar"></div>
        </section>
      </div>
    </section>
  </main>

  <script src="app.js"></script>
</body>
</html>
```

---

## 6. Styles: `site/styles.css`

```css
:root {
  --bg: #101010;
  --panel: #171717;
  --panel-light: #202020;
  --line: #353535;
  --text: #f5f3ef;
  --muted: #a8a8a8;
  --accent: #d9b77f;
  --white: #ffffff;
}

* {
  box-sizing: border-box;
}

html,
body {
  margin: 0;
  min-height: 100%;
  background: var(--bg);
  color: var(--text);
  font-family: Arial, Helvetica, sans-serif;
}

/* Hero page */

.hero-page {
  overflow: hidden;
}

.hero {
  position: relative;
  width: 100vw;
  height: 100vh;
  min-height: 520px;
  display: grid;
  place-items: center;
  overflow: hidden;
}

.hero-video {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.hero-overlay {
  position: absolute;
  inset: 0;
  background:
    linear-gradient(90deg, rgba(0, 0, 0, 0.7), rgba(0, 0, 0, 0.2)),
    linear-gradient(0deg, rgba(0, 0, 0, 0.4), transparent 55%);
}

.hero-content {
  position: relative;
  z-index: 1;
  width: min(900px, calc(100% - 48px));
  text-align: center;
}

.hero-kicker,
.panel-label,
.product-position {
  margin: 0 0 12px;
  color: var(--accent);
  font-size: 0.72rem;
  font-weight: bold;
  letter-spacing: 0.16em;
  text-transform: uppercase;
}

.hero h1 {
  max-width: 800px;
  margin: 0 auto;
  font-size: clamp(3rem, 8vw, 7rem);
  font-weight: 500;
  letter-spacing: -0.05em;
  line-height: 0.95;
}

.hero-description {
  max-width: 500px;
  margin: 24px auto 34px;
  color: #e5e5e5;
  font-size: 1.05rem;
  line-height: 1.6;
}

.hero-button {
  display: inline-block;
  padding: 15px 25px;
  border: 1px solid var(--white);
  color: var(--white);
  font-size: 0.8rem;
  font-weight: bold;
  letter-spacing: 0.1em;
  text-decoration: none;
  text-transform: uppercase;
  transition: 0.2s ease;
}

.hero-button:hover {
  background: var(--white);
  color: #111;
}

/* Catalogue layout */

.catalogue-page {
  min-width: 320px;
}

.catalogue-header {
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 24px;
  padding: 0 28px;
  border-bottom: 1px solid var(--line);
  background: #111;
}

.brand {
  color: var(--text);
  font-size: 0.75rem;
  font-weight: bold;
  letter-spacing: 0.12em;
  text-decoration: none;
}

.current-category {
  color: var(--muted);
  font-size: 0.85rem;
}

.catalogue-layout {
  display: grid;
  grid-template-columns: minmax(210px, 280px) 1fr;
  min-height: calc(100vh - 64px);
}

.category-panel {
  padding: 28px 16px;
  border-right: 1px solid var(--line);
  background: #131313;
}

.category-list {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.category-button {
  width: 100%;
  padding: 13px 14px;
  border: 1px solid transparent;
  border-radius: 0;
  background: transparent;
  color: var(--muted);
  cursor: pointer;
  font: inherit;
  font-size: 0.92rem;
  text-align: left;
  transition: 0.18s ease;
}

.category-button:hover {
  background: var(--panel-light);
  color: var(--white);
}

.category-button.active {
  border-color: var(--accent);
  background: #29231a;
  color: var(--white);
}

.product-panel {
  min-width: 0;
  background: var(--bg);
}

.product-content {
  min-height: calc(100vh - 64px);
  display: flex;
  flex-direction: column;
}

.product-main {
  display: grid;
  grid-template-columns: minmax(0, 1.35fr) minmax(310px, 0.65fr);
  flex: 1;
  min-height: 520px;
}

.image-area {
  position: relative;
  display: grid;
  min-height: 420px;
  place-items: center;
  padding: 36px;
  background: #e9e7e2;
}

.image-area img {
  width: 100%;
  height: 100%;
  max-height: calc(100vh - 250px);
  object-fit: contain;
}

.no-image-message {
  color: #4e4e4e;
  font-size: 0.9rem;
}

.product-information {
  overflow-y: auto;
  padding: 42px;
  background: var(--panel);
}

.product-information h1 {
  margin: 0 0 30px;
  font-size: clamp(1.8rem, 3vw, 3rem);
  font-weight: 500;
  letter-spacing: -0.04em;
  line-height: 1.05;
}

.metadata-list {
  margin: 0;
}

.metadata-list div {
  display: grid;
  grid-template-columns: minmax(105px, 0.38fr) 1fr;
  gap: 16px;
  padding: 13px 0;
  border-top: 1px solid var(--line);
}

.metadata-list dt {
  color: var(--muted);
  font-size: 0.78rem;
}

.metadata-list dd {
  margin: 0;
  color: var(--text);
  font-size: 0.9rem;
  line-height: 1.5;
  white-space: pre-wrap;
}

.product-navigation {
  display: flex;
  gap: 10px;
  margin-top: 34px;
}

.nav-button {
  flex: 1;
  padding: 13px 15px;
  border: 1px solid var(--line);
  background: transparent;
  color: var(--text);
  cursor: pointer;
  font-size: 0.78rem;
  font-weight: bold;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  transition: 0.18s ease;
}

.nav-button:hover:not(:disabled) {
  border-color: var(--accent);
  color: var(--accent);
}

.nav-button:disabled {
  cursor: not-allowed;
  opacity: 0.35;
}

.thumbnail-panel {
  width: 100%;
  padding: 18px 24px 22px;
  border-top: 1px solid var(--line);
  background: #151515;
}

.thumbnail-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
}

.thumbnail-header .panel-label {
  margin-bottom: 14px;
}

#thumbnailCount {
  color: var(--muted);
  font-size: 0.78rem;
}

.thumbnail-bar {
  display: flex;
  gap: 12px;
  overflow-x: auto;
  padding-bottom: 4px;
}

.product-thumbnail {
  width: 120px;
  flex: 0 0 120px;
  padding: 0;
  border: 1px solid transparent;
  background: transparent;
  color: var(--text);
  cursor: pointer;
  text-align: left;
}

.product-thumbnail.active {
  border-color: var(--accent);
}

.thumbnail-image {
  display: grid;
  width: 100%;
  height: 85px;
  place-items: center;
  overflow: hidden;
  background: #e9e7e2;
}

.thumbnail-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.thumbnail-image span {
  color: #666;
  font-size: 0.72rem;
}

.thumbnail-name {
  display: block;
  overflow: hidden;
  padding: 7px 6px;
  color: var(--muted);
  font-size: 0.72rem;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.product-thumbnail.active .thumbnail-name {
  color: var(--white);
}

.empty-state {
  padding: 80px 40px;
  text-align: center;
}

.empty-state h2 {
  margin: 0 0 12px;
  font-weight: 500;
}

.empty-state p {
  color: var(--muted);
}

.empty-state code {
  color: var(--accent);
}

/* Responsive layout */

@media (max-width: 900px) {
  .product-main {
    grid-template-columns: 1fr;
  }

  .image-area img {
    max-height: 55vh;
  }

  .product-information {
    padding: 30px;
  }
}

@media (max-width: 640px) {
  .catalogue-header {
    height: auto;
    align-items: flex-start;
    flex-direction: column;
    gap: 8px;
    padding: 18px;
  }

  .catalogue-layout {
    display: block;
  }

  .category-panel {
    border-right: 0;
    border-bottom: 1px solid var(--line);
    padding: 18px;
  }

  .category-list {
    flex-direction: row;
    overflow-x: auto;
  }

  .category-button {
    width: auto;
    flex: 0 0 auto;
  }

  .product-main {
    min-height: auto;
  }

  .image-area {
    min-height: 340px;
    padding: 20px;
  }

  .thumbnail-panel {
    padding: 18px;
  }
}
```

---

## 7. Catalogue JavaScript: `site/app.js`

```javascript
const state = {
  categories: [],
  categoryIndex: 0,
  productIndex: 0
};

const elements = {
  categoryList: document.getElementById("categoryList"),
  categoryTitle: document.getElementById("categoryTitle"),
  productContent: document.getElementById("productContent"),
  emptyState: document.getElementById("emptyState"),
  productImage: document.getElementById("productImage"),
  noImageMessage: document.getElementById("noImageMessage"),
  productName: document.getElementById("productName"),
  productPosition: document.getElementById("productPosition"),
  metadataList: document.getElementById("metadataList"),
  previousButton: document.getElementById("previousButton"),
  nextButton: document.getElementById("nextButton"),
  thumbnailBar: document.getElementById("thumbnailBar"),
  thumbnailCount: document.getElementById("thumbnailCount")
};

function getCurrentCategory() {
  return state.categories[state.categoryIndex];
}

function getCurrentProduct() {
  const category = getCurrentCategory();
  return category?.products[state.productIndex];
}

function formatValue(value) {
  if (value === null || value === undefined || value === "") {
    return "—";
  }

  if (Array.isArray(value)) {
    return value.join(", ");
  }

  if (typeof value === "object") {
    return JSON.stringify(value, null, 2);
  }

  return String(value);
}

function renderCategories() {
  elements.categoryList.replaceChildren();

  state.categories.forEach((category, index) => {
    const button = document.createElement("button");

    button.type = "button";
    button.className = "category-button";
    button.textContent = category.name;

    if (index === state.categoryIndex) {
      button.classList.add("active");
    }

    button.addEventListener("click", () => {
      state.categoryIndex = index;
      state.productIndex = 0;
      renderCatalogue();
    });

    elements.categoryList.appendChild(button);
  });
}

function renderMetadata(metadata) {
  elements.metadataList.replaceChildren();

  const entries = Object.entries(metadata || {});

  if (entries.length === 0) {
    const row = document.createElement("div");
    const label = document.createElement("dt");
    const value = document.createElement("dd");

    label.textContent = "Information";
    value.textContent = "No product metadata available.";

    row.append(label, value);
    elements.metadataList.appendChild(row);
    return;
  }

  entries.forEach(([key, value]) => {
    const row = document.createElement("div");
    const label = document.createElement("dt");
    const content = document.createElement("dd");

    label.textContent = key;
    content.textContent = formatValue(value);

    row.append(label, content);
    elements.metadataList.appendChild(row);
  });
}

function renderThumbnails(category) {
  elements.thumbnailBar.replaceChildren();
  elements.thumbnailCount.textContent = `${category.products.length} product${category.products.length === 1 ? "" : "s"}`;

  category.products.forEach((product, index) => {
    const button = document.createElement("button");
    const imageContainer = document.createElement("span");
    const name = document.createElement("span");

    button.type = "button";
    button.className = "product-thumbnail";

    if (index === state.productIndex) {
      button.classList.add("active");
    }

    imageContainer.className = "thumbnail-image";
    name.className = "thumbnail-name";
    name.textContent = product.name;

    if (product.images?.length) {
      const image = document.createElement("img");
      image.src = product.images[0];
      image.alt = product.name;
      image.loading = "lazy";
      imageContainer.appendChild(image);
    } else {
      const noImage = document.createElement("span");
      noImage.textContent = "No image";
      imageContainer.appendChild(noImage);
    }

    button.append(imageContainer, name);

    button.addEventListener("click", () => {
      state.productIndex = index;
      renderCatalogue();

      window.scrollTo({
        top: 0,
        behavior: "smooth"
      });
    });

    elements.thumbnailBar.appendChild(button);
  });
}

function renderProduct() {
  const category = getCurrentCategory();
  const product = getCurrentProduct();

  if (!category || !product) {
    elements.productContent.hidden = true;
    elements.emptyState.hidden = false;
    elements.categoryTitle.textContent = "No products available";
    return;
  }

  elements.productContent.hidden = false;
  elements.emptyState.hidden = true;

  elements.categoryTitle.textContent = category.name;
  elements.productName.textContent = product.name;
  elements.productPosition.textContent =
    `Product ${state.productIndex + 1} of ${category.products.length}`;

  const firstImage = product.images?.[0];

  if (firstImage) {
    elements.productImage.src = firstImage;
    elements.productImage.alt = product.name;
    elements.productImage.hidden = false;
    elements.noImageMessage.hidden = true;
  } else {
    elements.productImage.removeAttribute("src");
    elements.productImage.alt = "";
    elements.productImage.hidden = true;
    elements.noImageMessage.hidden = false;
  }

  renderMetadata(product.metadata);
  renderThumbnails(category);

  elements.previousButton.disabled = state.productIndex === 0;
  elements.nextButton.disabled =
    state.productIndex === category.products.length - 1;
}

function renderCatalogue() {
  renderCategories();
  renderProduct();
}

function moveProduct(direction) {
  const category = getCurrentCategory();
  const nextIndex = state.productIndex + direction;

  if (!category || nextIndex < 0 || nextIndex >= category.products.length) {
    return;
  }

  state.productIndex = nextIndex;
  renderCatalogue();

  window.scrollTo({
    top: 0,
    behavior: "smooth"
  });
}

elements.previousButton.addEventListener("click", () => moveProduct(-1));
elements.nextButton.addEventListener("click", () => moveProduct(1));

async function loadCatalogue() {
  try {
    const response = await fetch("/api/catalogue");

    if (!response.ok) {
      throw new Error("Catalogue data could not be loaded.");
    }

    const data = await response.json();

    state.categories = (data.categories || []).filter(
      (category) => category.products?.length > 0
    );

    renderCatalogue();
  } catch (error) {
    elements.productContent.hidden = true;
    elements.emptyState.hidden = false;
    elements.emptyState.innerHTML = `
      <h2>Catalogue could not be loaded</h2>
      <p>Start the catalogue using <code>python server.py</code>.</p>
    `;

    elements.categoryTitle.textContent = "Catalogue unavailable";
    console.error(error);
  }
}

loadCatalogue();
```

---

## 8. Run the Catalogue

Open a terminal in the main `offline_catalogue` folder and run:

```bash
python server.py
```

For some systems, use:

```bash
python3 server.py
```

Then open:

```text
http://127.0.0.1:8000
```

---

## Important Notes

- Do **not** open `index.html` directly by double-clicking it, because browsers may block local JavaScript requests to product data.
- The Python server works fully **offline**. It only uses your own computer at `127.0.0.1`.
- Add or remove product folders, then refresh the browser. The catalogue is rebuilt automatically from the folder structure.
- Keep product image filenames and names in `product.json` exactly matched. Case differences matter on some operating systems—tiny filename gremlins are surprisingly powerful.

```
