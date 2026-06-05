# Wtr-Lab Jetpack Compose UI Subsystem Manual
## UI Layouts, Interactive Components, and WebView Scraping Drivers

This document details the screens, sheets, dynamic Compose trees, and injected styles framing the Wtr-Lab web novel e-reading experience.

---

## 🎨 Design System, Colors, and Themes

The visual theme utilizes standard **Material Design 3 (M3)** schemas with custom dark options optimized for eye-safe night reading.
- **Edge-to-Edge Constraints**: Standard programmatic top and bottom margin offsets using `WindowInsets.systemBars` are injected straight into custom `Scaffold` elements to support edge-to-edge layouts natively.
- **Touch Targets**: All control surfaces (Play, Pause, Track Bar, Tabs grids) maintain standard **48dp minimum physical heights** to adhere strictly to Material accessibility recommendations.

---

## 📱 BrowserAppScreen.kt (The Main Container View)

`BrowserAppScreen` presents the core interface container of the application. It acts as a multi-layered viewport displaying active WebViews, search components, custom Speed Dials, and the background audio shelf overlay.

```
+-------------------------------------------------------------+
|  [Address & URL Search Bar with Tab Toggle Counter]         |
+-------------------------------------------------------------+
|                                                             |
|  [Active Tab WebView Instance (Pooled)]                     |
|                                                             |
|  - Renders Active Chapter Contents                          |
|  - Custom JS Injection Engine Highlights current paragraph  |
|                                                             |
+-------------------------------------------------------------+
|  [Floating Action Button : TTS Play Indicator Overlay]      |
+-------------------------------------------------------------+
|  [Bottom Audiobook Controller Shelf]                        |
|  - Play / Pause | Seek Bar | Chapter Skip                 |
+-------------------------------------------------------------+
```

### 1. Injected Stylesheet Rules (Eye-Safe Night Reading)
Applying dark-mode modifications directly onto standard novel pages forces custom CSS stylesheets straight into client rendering loops:
```javascript
let darkStyle = document.createElement('style');
darkStyle.innerHTML = 'body, div, p, span, article, section { background-color: #121212 !important; color: #E0E0E0 !important; }';
document.head.appendChild(darkStyle);
```

---

## 📚 Dynamic Paragraph Web Scraper (`runHtmlTextExtractionAndPlay`)

One of the application's most critical subsystems is its **custom paragraph DOM extractor**, optimized extensively to handle infinite vertical scroll layouts like `webnovel.com`.

### 1. Unified Web Scraper Core Logic (JavaScript Implementation)

```javascript
(function() {
    let paragraphs = [];
    let elements = [];
    let host = window.location.hostname;
    
    // Junk filter keyword dictionary
    function isJunk(text) {
        let t = text.toLowerCase().trim();
        if (t.length < 5) return true;
        if (t.includes(".com") || t.includes(".org") || t.includes(".net") || t.includes(".me") || t.includes(".xyz") || t.includes("http://") || t.includes("https://")) {
            if (t.includes("novelbin") || t.includes("novelhall") || t.includes("freewebnovel") || t.includes("fanmtl") || t.includes("timotxt") || t.includes("novel543") || t.includes("twkan") || t.includes("novelhub") || t.includes("novelhubapp") || t.includes("webnovel")) {
                return true;
            }
        }
        const promoKeywords = [
            "join our discord", "patreon", "support me", "recommend", "rate this", "novelbin", "novelhall", "freewebnovel", "fanmtl", "timotxt", "novel543", "twkan", "novelhub", "novelhubapp"
        ];
        return promoKeywords.some(keyword => t.includes(keyword));
    }
    
    let rawContainers = Array.from(document.querySelectorAll('.cha-content, .chapter-content, .cha-words, .chapter-inner'));
    
    // Anti-overlapping sibling filter logic:
    // Guarantees only top-level parents are selected; filters out nested duplicate components!
    let containers = rawContainers.filter(c => !rawContainers.some(other => other !== c && other.contains(c)));
    
    containers.forEach(contentEl => {
        let pSelector = host.includes("webnovel") ? 'p, .cha-paragraph, .pirate' : 'p, .wtr-line-segment';
        let rawPTags = Array.from(contentEl.querySelectorAll(pSelector));
        let pTags = rawPTags.filter(p => !rawPTags.some(parent => parent !== p && parent.contains(p)));
        
        pTags.forEach(p => {
            let excludeClass = '.author-note, .gift-box, .recommend-box, .comment-area, .m-comment, .user-opinion';
            if (!p.closest(excludeClass)) {
                let rect = p.getBoundingClientRect();
                let isVisible = rect.height > 0 || p.offsetHeight > 0;
                let text = p.innerText.trim();
                
                // Excludes invisible spam elements and advertising scripts 
                if (text.length > 5 && isVisible && !isJunk(text)) {
                    paragraphs.push(text);
                }
            }
        });
    });
    
    return JSON.stringify(paragraphs);
})();
```

---

## 🎨 Bottom Audiobook Controller Shelf UI

Renders playback tracks sequentially using standard audio controls:
- Displays novel name titles, current chapter labels, and source host URLs.
- Houses standard **Play / Pause**, **Skip Previous**, **Skip Next**, and **Playback Speed** buttons.
- Draws in-line progress indicators tracking TTS completion sequences (e.g., `Paragraph 12 of 88`).
- Binds direct callbacks to trigger lockscreen notifications or page-turn requests.

---

## 📑 SettingsDialog.kt (The Diagnostics Hub)

Houses controls managing localized runtime configurations:
- **TTS Settings Slider**: Custom sliders tweaking pitch and rate multipliers (e.g. `0.5x - 2.5x`).
- **Force Dark Toggle**: Installs or pulls eye-safe dark stylesheets dynamically.
- **Session Log Viewer**: Exposes scrolling text logs of the thread-safe `WtrLogManager` history records directly to inside-app debug panels.
- **Cache/Cookie Sweeper**: Drops client-side database records to release app memory allocations safely.

---

## 🏠 ChromeNewTabPage.kt (The Speed Dial Page)

Drawn when a tab is pointing to `chrome://newtab`.
- Integrates custom search inputs forwarding queries to search providers or resolving URL redirects.
- Standard Grid Cards displaying quick navigation tiles (Google, Webnovel, Royal Road, Scribble Hub).
- Recent History view: Iterates over the `visitedAt` database list to display quick-jump anchor items.
