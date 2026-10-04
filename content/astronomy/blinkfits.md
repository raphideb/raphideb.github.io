---
title: "BlinkFits"
linkTitle: "BlinkFits"
description: "A free, lightweight FITS viewer for Windows. Compare frames and automatically find satellite trails, blurry captures or passing clouds."
type: "docs"
---

<style>
.bf {
  --panel: #F6F7F9;
  --raised: #EEF0F3;
  --border: #DDE1E6;
  --text: #1B1F24;
  --dim: #5A636E;
  --accent: #1F6FEB;
  --accent-text: #FFFFFF;
  --mark: #C26A00;
  --code: #F3F4F6;
  --sky: #050607;
  --star: #E6E8EB;
  --sky-mark: #F0A53E;
  --btn: var(--accent);
  color: var(--text);
}
/* Docsy sets data-bs-theme on <html> when switching to dark (also for auto). */
html[data-bs-theme="dark"] .bf {
  --panel: #2B3035;
  --raised: #343A40;
  --border: #495057;
  --text: #DEE2E6;
  --dim: #A7B0BA;
  --accent: #6EA8FE;
  --mark: #F0A53E;
  --code: #1A1D20;
  --btn: #1F6FEB;
}
.bf a { color: var(--accent); }
.bf code, .bf kbd, .bf .mono { font-family: ui-monospace, "Cascadia Mono", "Cascadia Code", Consolas, "SF Mono", Menlo, monospace; }
.bf code {
  background: var(--code); color: var(--text);
  border: 1px solid var(--border); border-radius: 4px;
  padding: .05em .35em; font-size: .9em;
}
.bf kbd {
  display: inline-block; min-width: 1.7em; text-align: center;
  background: var(--raised); color: var(--text); box-shadow: none;
  border: 1px solid var(--border); border-bottom-width: 2px;
  border-radius: 5px; padding: 0 .4em; font-size: .85em; line-height: 1.6;
}
.bf .sub { color: var(--dim); max-width: 44em; }

.bf-hero { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 40px; align-items: center; margin: 8px 0 40px; }
.bf-brand { display: flex; align-items: center; gap: 10px; font-weight: 600; font-size: 1.15rem; margin: 0 0 16px; }
.bf-brand img { width: 32px; height: 32px; }
.bf-brand .fits { color: var(--accent); }
.bf-brand .ver { color: var(--dim); font-weight: 400; }
.bf .eyebrow { color: var(--mark); font-size: .85rem; letter-spacing: .08em; text-transform: uppercase; font-weight: 600; margin: 0 0 8px; }
.bf .headline { font-size: clamp(1.8rem, 3.4vw, 2.6rem); line-height: 1.12; font-weight: 700; margin: 0 0 16px; letter-spacing: -.01em; }
.bf .lead { color: var(--dim); font-size: 1.1rem; margin: 0 0 24px; }

.bf-download { background: var(--panel); border: 1px solid var(--border); border-radius: 12px; padding: 20px; }
.bf .btn-dl {
  display: flex; align-items: center; gap: 14px;
  background: var(--btn); color: var(--accent-text);
  text-decoration: none; border-radius: 8px; padding: 14px 20px;
  font-weight: 600; font-size: 1.08rem;
}
.bf .btn-dl:hover { filter: brightness(1.1); color: var(--accent-text); }
.bf .btn-dl svg { flex: none; }
.bf .btn-dl small { display: block; font-weight: 400; font-size: .85rem; opacity: .85; }
.bf .sha { margin-top: 16px; }
.bf .sha-label { display: flex; justify-content: space-between; align-items: baseline; font-size: .85rem; color: var(--dim); margin-bottom: 6px; }
.bf .sha-box { display: flex; align-items: stretch; background: var(--code); border: 1px solid var(--border); border-radius: 6px; overflow: hidden; }
.bf .sha-box code { flex: 1; border: 0; border-radius: 0; background: none; padding: 9px 12px; font-size: .8rem; line-height: 1.5; word-break: break-all; }
.bf .copy {
  flex: none; background: var(--raised); color: var(--text);
  border: 0; border-left: 1px solid var(--border);
  padding: 0 14px; font: inherit; font-size: .85rem; cursor: pointer;
}
.bf .copy:hover { color: var(--accent); }
.bf .hint { font-size: .82rem; color: var(--dim); margin: 8px 0 0; }
.bf .facts { display: flex; flex-wrap: wrap; gap: 6px 18px; font-size: .88rem; color: var(--dim); margin: 16px 0 0; padding: 0; list-style: none; }
.bf .facts li { margin: 0; }
.bf .facts li::before { content: "✓"; color: var(--mark); margin-right: 6px; }

.bf .frame { position: relative; background: var(--sky); border: 1px solid var(--border); border-radius: 10px; overflow: hidden; aspect-ratio: 16 / 10; }
.bf .frame svg { display: block; width: 100%; height: 100%; }
.bf .frame-bar {
  position: absolute; left: 0; right: 0; top: 0; display: flex; justify-content: space-between; align-items: center; gap: 12px;
  padding: 8px 12px; font-size: .78rem; color: #9098A0; background: rgba(10, 11, 13, .75);
}
.bf .frame-bar .name { color: #E6E8EB; }
.bf .frame.bad .frame-bar .name { color: var(--sky-mark); }
.bf .frame .tag {
  display: none; margin-right: 10px; padding: 0 6px; border: 1px solid var(--sky-mark); border-radius: 3px;
  color: var(--sky-mark); font-size: .72rem; letter-spacing: .06em; text-transform: uppercase;
}
.bf .frame.bad .tag { display: inline-block; }
.bf .frame .trail { opacity: 0; }
.bf .frame.is-trail .trail { opacity: 1; }
.bf .blink-caption { font-size: .88rem; color: var(--dim); margin: 12px 0 0; }
.bf .blink-caption b { color: var(--mark); font-weight: 600; }

.bf-news { background: var(--panel); border: 1px solid var(--border); border-left: 3px solid var(--accent); border-radius: 12px; padding: 20px 24px; margin: 0 0 40px; }
.bf-news .head { display: flex; flex-wrap: wrap; align-items: baseline; gap: 4px 12px; margin: 0 0 14px; font-size: 1.02rem; font-weight: 600; }
.bf-news .date { font-size: .85rem; font-weight: 400; color: var(--dim); }
.bf-news dl { margin: 0; }
.bf-news dt { font-size: .95rem; font-weight: 600; margin: 0; }
.bf-news dd { margin: 2px 0 12px; color: var(--dim); font-size: .93rem; }
.bf-news dd:last-child { margin-bottom: 0; }
.bf-news ul { margin: 6px 0 0; padding-left: 1.2em; }
.bf-news li { margin: 2px 0; }

.bf .shot { margin: 0 0 40px; }
.bf .shot img { display: block; width: 100%; height: auto; border: 1px solid var(--border); border-radius: 10px; }
.bf .shot figcaption { font-size: .88rem; color: var(--dim); margin-top: 12px; }

.bf .steps { max-width: 44em; margin-bottom: 40px; }

.bf .grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 16px; margin-bottom: 40px; }
.bf .card { background: var(--panel); border: 1px solid var(--border); border-radius: 10px; padding: 20px; }
.bf .card h3 { margin: 0 0 6px; font-size: 1.02rem; display: flex; align-items: center; gap: 10px; }
.bf .card h3::before { content: ""; width: 8px; height: 8px; border-radius: 50%; background: var(--accent); flex: none; }
.bf .card.hl h3::before { background: var(--mark); }
.bf .card p, .bf .card ul { margin: 0 0 8px; color: var(--dim); font-size: .95rem; }
.bf .card ul { padding-left: 1.2em; }
.bf .card li { margin: 2px 0; }
.bf .card > :last-child { margin-bottom: 0; }

.bf .two { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 40px; margin-bottom: 40px; }
.bf .two table { width: 100%; font-size: .95rem; }
.bf .two td:first-child { white-space: nowrap; padding-right: 20px; }

.bf .note { background: var(--panel); border: 1px solid var(--border); border-left: 3px solid var(--mark); border-radius: 8px; padding: 18px 20px; margin-bottom: 40px; }
.bf .note p, .bf .note ul, .bf .note ol { margin: 0 0 10px; color: var(--dim); }
.bf .note ul, .bf .note ol { padding-left: 1.4em; }
.bf .note > :last-child { margin: 0; }
.bf .note strong { color: var(--text); }

.bf .credits { border-top: 1px solid var(--border); padding-top: 24px; font-size: .88rem; color: var(--dim); }
.bf .credits p { margin: 0 0 6px; }
</style>

<div class="bf">

<div class="bf-hero">
<div>
<div class="bf-brand"><img src="/astronomy/blinkfits-icon.png" alt="" width="32" height="32"><span>Blink<span class="fits">Fits</span> <span class="ver">1.3</span></span></div>
<p class="eyebrow">Lightweight FITS viewer for Windows</p>
<p class="headline">Find bad frames, stack only the good ones.</p>
<p class="lead">BlinkFits can auto-detect and mark frames with satellite trails, soft stars or passing clouds. Blink through your frames with zoom, pan and stretch held perfectly still. Throw bad subs out before you stack and get only the best data.</p>
<div class="bf-download">
<a class="btn-dl" href="https://github.com/raphideb/blinkfits_release/releases/latest/download/blinkfits.zip">
<svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M12 3v12"/><path d="m6 10 6 6 6-6"/><path d="M4 20h16"/></svg>
<span>Download latest version for Windows<small>blinkfits.zip</small></span>
</a>
<p class="hint">Extract the zip, then run <code>blinkfits.exe</code>. No installer needed.</p>
<p class="hint">Need an older version? <a href="https://github.com/raphideb/blinkfits_release/releases">All releases are on GitHub</a>.</p>
<div class="sha">
<div class="sha-label"><span>SHA-256 of <code>blinkfits.exe</code>, version 1.3</span><span id="bf-copied" aria-live="polite"></span></div>
<div class="sha-box"><code id="bf-check">c383a2e6b1a00365e9312c15429e8639afece3c4582ea3de66ff8e1ef617b32b</code><button class="copy" id="bf-copy" type="button">Copy</button></div>
<p class="hint">How to check the exe against it, and what to do when Windows warns you: <a href="#download-help">see below</a>.</p>
</div>
<ul class="facts">
<li>Free</li>
<li>Windows 10 / 11, 64 bit</li>
<li>No installer</li>
<li>Offline, no telemetry</li>
</ul>
</div>
</div>
<div class="blink" role="img" aria-label="Animated example: BlinkFits blinks through five frames of a star field. One frame has a satellite trail and one has blurred stars. Both are tagged and renamed.">
<div class="frame" id="bf-demo">
<svg viewBox="0 0 640 400" preserveAspectRatio="xMidYMid slice" aria-hidden="true">
<defs>
<filter id="bf-soft" x="-10%" y="-10%" width="120%" height="120%">
<feGaussianBlur stdDeviation="2.6"/>
<feComponentTransfer><feFuncA type="linear" slope="2.4"/></feComponentTransfer>
</filter>
</defs>
<g id="bf-field"><g id="bf-stars"></g></g>
<g class="trail">
<line x1="-20" y1="96" x2="660" y2="318" stroke="var(--star)" stroke-width="6" opacity=".12"/>
<line x1="-20" y1="96" x2="660" y2="318" stroke="var(--star)" stroke-width="1.5" opacity=".9"/>
</g>
</svg>
<div class="frame-bar mono"><span class="name" id="bf-demo-name">frame_00041.fits</span><span><span class="tag" id="bf-demo-tag"></span><span id="bf-demo-count">41 / 218</span></span></div>
</div>
<p class="blink-caption">The view never moves, so a trail or a soft frame jumps out at once. <b>BlinkFits finds them for you</b> and tags the files.</p>
</div>
</div>

<div class="bf-news">
<p class="head">New version 1.3 released!<span class="date">4 October 2026</span></p>
<dl>
<dt>Cloud detection</dt>
<dd><b>Find blur</b> now also catches frames where a thin cloud drifted past: with <b>Detect clouds</b> checked, it tags every frame whose sky is brighter or more patchy than in the frames taken just before and after it.</dd>
<dt>Selecting with the keyboard</dt>
<dd><kbd>Shift</kbd> with <kbd>↑</kbd> or <kbd>↓</kbd> now starts a fresh selection at the current image once you have moved on, and <kbd>Ctrl</kbd> + <kbd>Shift</kbd> with <kbd>↑</kbd> or <kbd>↓</kbd> adds a new range to the selection you already have.</dd>
</dl>
</div>

<figure class="shot">
<img src="/astronomy/blinkfits-screenshot.jpg" width="1600" height="865" loading="lazy" alt="BlinkFits with 611 raw frames of M 31 loaded and Show blur on: the list on the left holds only the 267 frames tagged _blur, the middle shows frame 13 of 267, where a thin cloud has washed out the sky around the galaxy, and the right panel shows the Find bad images section with Allowed blur deviation at 50%, Detect clouds checked and Trails: 0 · Blur: 267">
<figcaption>BlinkFits with 611 frames of M 31 from a night of passing cloud. Find blur has tagged 267 of them. With Detect clouds checked, it tags every frame whose sky is brighter or more patchy than in the frames around it, even when its stars are still sharp. Show blur narrows the list and blinks through those frames, so you can check each one before you move it out of the stack.</figcaption>
</figure>

## From raw subs to a clean stack {#workflow}

<div class="steps">

A single satellite trail or a handful of soft frames can cost detail in the final image. Pixel rejection in the stacker catches some of it, but not all. BlinkFits lets you look at every sub and take out the bad ones first.

A night's work goes like this:

1. Press **Add folder** and add the folders of the night.
2. Press **Find trails** and **Find blur**. They tag the obvious rejects for you.
3. Blink through the rest. Press <kbd>M</kbd> on anything the filters missed, such as a guiding error.
4. Press <kbd>X</kbd> and pick a folder. Every marked and tagged frame moves there.
5. Stack what is left.

</div>

## Check your subs and throw out the bad ones {#features}

<div class="grid">
<div class="card hl">
<h3>Let it find the bad subs</h3>
<p>With hundreds of subs, going through them one by one takes a long time. Two filters do the first pass:</p>
<ul>
<li><b>Find trails</b> tags frames crossed by a satellite or an aircraft.</li>
<li><b>Find blur</b> tags frames whose stars are bloated or gone, compared with the sharpest frame. With <b>Allowed blur deviation</b> you set how much larger the stars may be. With <b>Detect clouds</b> checked, it also tags frames whose sky a passing cloud has brightened or made patchy.</li>
</ul>
<p>A tag renames the file: <code>frame.fits</code> becomes <code>frame_trail.fits</code> or <code>frame_blur.fits</code>. <b>Show trails</b> and <b>Show blur</b> blink through just those frames, so you can check every find.</p>
<p>If a filter got one wrong, press <kbd>M</kbd> on it and the tag comes off.</p>
</div>
<div class="card hl">
<h3>Mark what the filters missed</h3>
<p>Some bad frames need your eye, for example a guiding error. Press <kbd>M</kbd>, and the file is renamed on disk with your suffix: <code>frame.fits</code> becomes <code>frame_bad.fits</code>.</p>
<p>The mark lives in the file name, so it is still there next session and in Explorer. Select several frames to mark them in one go. <b>Show marked</b> blinks through only the marked frames.</p>
</div>
<div class="card hl">
<h3>Move them out of the stack</h3>
<p><b>Move marked to folder</b>, or <kbd>X</kbd>, moves every marked and tagged frame into a folder you choose. What stays behind is ready to stack.</p>
<p>The dialog opens in the folder you used last. Each moved file gets a number, so that frames with the same name from different folders never overwrite each other.</p>
</div>
<div class="card">
<h3>The view never moves</h3>
<p>Zoom, pan and stretch belong to the viewer, not to the frame. When you switch images, only the sky changes. That is what makes a trail or a soft frame jump out.</p></div>
<div class="card">
<h3>Auto blink</h3>
<p>Auto blink runs through the frames on its own, at 0.1 s to 60 s per frame, and wraps round at both ends. You steer it like this:</p>
<ul>
<li><kbd>Space</kbd> or <b>Pause</b> stops it, and it resumes at the same speed.</li>
<li><b>Reverse</b> runs it backwards.</li>
<li><kbd>←</kbd> and <kbd>→</kbd> turn it round while it runs.</li>
</ul>
</div>
<div class="card">
<h3>Six stretches</h3>
<p>A strong stretch shows faint trails and soft stars that a linear view hides. Auto (STF) is the default. Levels, Asinh, Logarithmic, Histogram equalisation and Unstretched are one click away.</p>
<p>The sliders react instantly, even on large frames.</p>
</div>
<div class="card">
<h3>Past the meridian flip</h3>
<p>After a meridian flip, the frames are upside down, which makes blinking the whole night hard. Select the frames taken after the flip and press <kbd>R</kbd> to turn them by 180°.</p>
<p>Only the display is turned. The file is never written, and a debayered frame keeps its colours.</p>
</div>
<div class="card">
<h3>One shot colour</h3>
<p>Raw OSC frames are debayered from <code>BAYERPAT</code> in the header. You can also pick RGGB, BGGR, GRBG or GBRG yourself. Each channel is stretched on its own, for a neutral sky.</p>
</div>
<div class="card">
<h3>FITS header</h3>
<p>When one frame looks off, the header often tells you why. <b>FITS header</b> shows the header cards in place of the image.</p>
<p>Keep blinking and the header follows, frame by frame: exposure, gain, temperature, time and whatever else is in your header.</p>
</div>
<div class="card">
<h3>Zoom in</h3>
<p><b>Zoom</b> shows a frame at 5 to 100 percent, and <b>Magnify</b> carries on up to 10×, so you can compare frames pixel by pixel.</p>
<p>The mouse wheel runs through every step and zooms around the pointer.</p>
</div>
<div class="card">
<h3>Several folders</h3>
<p>Add files and folders from different sessions to one list, and remove any of them again. You can also pass paths on the command line.</p>
</div>
<div class="card">
<h3>Careful with memory</h3>
<p>BlinkFits keeps 16 bit samples and a cache that fits your RAM. When memory runs short, it shows a clear message instead of swapping your machine to a halt.</p>
</div>
<div class="card">
<h3>15 languages</h3>
<p>English, Deutsch, Français, Italiano, Español, Português (BR / PT), Nederlands, Dansk, Svenska, Polski, Русский, Українська, 日本語 and 中文.</p>
</div>
</div>

<div class="two">
<div>

## Hands on the keyboard {#keys}

<p class="sub">Checking hundreds of subs is fastest without the mouse.</p>

| Key | Action |
|-----|--------|
| <kbd>←</kbd> <kbd>→</kbd> &nbsp;or&nbsp; <kbd>A</kbd> <kbd>D</kbd> | Go to the previous or next image. While auto blink runs, these reverse its direction. |
| <kbd>Space</kbd> | Start or stop auto blink. |
| <kbd>Home</kbd> <kbd>End</kbd> | Jump to the first or last image. |
| <kbd>Shift</kbd> + <kbd>↑</kbd> <kbd>↓</kbd> | Start a new selection at the current image and extend it up or down. |
| <kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>↑</kbd> <kbd>↓</kbd> | Add a new range to the selection. |
| <kbd>Shift</kbd> + <kbd>Home</kbd> <kbd>End</kbd> | Select up to the first or last image of the folder. |
| <kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Home</kbd> <kbd>End</kbd> | Select up to the first or last image of the whole list. |
| <kbd>Ctrl</kbd> + <kbd>A</kbd> | Select all images. |
| <kbd>M</kbd> | Mark the current image, or every selected one. |
| <kbd>R</kbd> | Turn the current image, or every selected one, by 180°. |
| <kbd>X</kbd> | Move marked and tagged images to a folder. |
| <kbd>S</kbd> | Switch to the next stretch method. |
| <kbd>1</kbd> … <kbd>6</kbd> | Zoom to 5, 10, 25, 50, 75 or 100 %. |
| Mouse wheel | Zoom in or out around the pointer, from 5 % to 10×. |
| <kbd>F</kbd> or <kbd>0</kbd> | Fit the image to the window. |
| <kbd>N</kbd> | Turn night vision on or off. The interface turns dim red, the image stays as it is. |
| <kbd>F1</kbd> | Open the help. |
| <kbd>Esc</kbd> | Close the FITS header, Help or About. Otherwise, clear the selection. |

</div>
<div>

## Files it reads {#files}

<p class="sub">Straight from your capture software, no conversion.</p>

- **Uncompressed FITS** with BITPIX 8, 16, 32, 64, −32 or −64, including BZERO / BSCALE and BLANK.
- **Mono, raw Bayer and three plane colour** images. Raw frames stay grey unless Debayer is on.
- **Up to 200 MB per file.** Larger ones are refused with a message.
- **Not supported:** tile compressed FITS (`ZIMAGE`, e.g. `.fz`).

### What you need

- **Windows 10 or 11, 64 bit.** Nothing else: one exe, no installer, no runtime, no DLLs.
- Settings live in `%APPDATA%\blinkfits`. Delete the exe and that folder, and BlinkFits is gone.

</div>
</div>

## Checking the download {#download-help}

<p class="sub">BlinkFits is a hobby project, and the exe is not code signed. Windows treats new, unsigned programs with caution until enough people have run them.</p>

<div class="note">
<p><strong>Check the file.</strong> The download box above shows the SHA-256 checksum of the exe. If your copy has the same checksum, it arrived intact. After extracting, calculate it in one of two ways:</p>
<ul>
<li>In PowerShell: <code>Get-FileHash .\blinkfits.exe -Algorithm SHA256</code></li>
<li>In a command prompt: <code>certutil -hashfile blinkfits.exe SHA256</code></li>
</ul>
<p>Then compare the result with the checksum above.</p>
<p><strong>Windows SmartScreen</strong> may say “Windows protected your PC” when you start it. To run BlinkFits anyway:</p>
<ol>
<li>Click <em>More info</em>.</li>
<li>Click <em>Run anyway</em>.</li>
</ol>
</div>

<div class="two">
<div>

## Terms of use {#terms}

<p class="sub">Anyone may download and use BlinkFits free of charge, for any purpose. Redistributing, selling or modifying it, or passing it off as your own work, is not permitted without the author's written permission. It is provided as is, without any warranty.</p>

</div>
<div>

## Contact {#contact}

<p class="sub">Questions, bug reports, a FITS file that will not open? Write to <a href="mailto:raphi@crashdump.ch">raphi@crashdump.ch</a>.</p>
<p class="sub">Please share a link to this page rather than to the download. That way people always get the current version and its checksum.</p>
<p class="sub"><a href="https://paypal.me/RaphaelDebinski" target="_blank" rel="noopener">Support me with PayPal</a></p>

</div>
</div>

<div class="credits">
<p>© 2026 Raphael Debinski. All rights reserved.</p>
<p>Built with Go, Gio and go-text/typesetting. Their open source licenses are shown in the program under About and apply to those components only.</p>
</div>

</div>

<script>
(function () {
  // Copy the checksum.
  var copy = document.getElementById("bf-copy"), hash = document.getElementById("bf-check"), said = document.getElementById("bf-copied");
  copy.addEventListener("click", function () {
    var text = hash.textContent.trim();
    function done() { said.textContent = "Copied"; copy.textContent = "✓"; setTimeout(function () { said.textContent = ""; copy.textContent = "Copy"; }, 1600); }
    if (navigator.clipboard && window.isSecureContext) {
      navigator.clipboard.writeText(text).then(done, select);
    } else { select(); }
    function select() {
      var r = document.createRange(); r.selectNodeContents(hash);
      var s = window.getSelection(); s.removeAllRanges(); s.addRange(r);
      try { if (document.execCommand("copy")) done(); } catch (e) {}
    }
  });

  // Star field for the blink demo, the same every visit.
  var seed = 7331;
  function rnd() { seed = (seed * 16807) % 2147483647; return (seed - 1) / 2147483646; }
  var g = document.getElementById("bf-stars"), out = [];
  for (var i = 0; i < 170; i++) {
    var x = rnd() * 640, y = 30 + rnd() * 370, m = Math.pow(rnd(), 5);
    var r = 0.6 + m * 2.6, o = 0.35 + rnd() * 0.45 + m * 0.3;
    out.push('<circle cx="' + x.toFixed(1) + '" cy="' + y.toFixed(1) + '" r="' + r.toFixed(2) +
             '" fill="var(--star)" opacity="' + Math.min(o, 1).toFixed(2) + '"/>');
  }
  // A faint galaxy smudge, because every field has one.
  out.push('<ellipse cx="470" cy="150" rx="22" ry="7" transform="rotate(-28 470 150)" fill="var(--star)" opacity=".16"/>');
  out.push('<ellipse cx="470" cy="150" rx="9" ry="3" transform="rotate(-28 470 150)" fill="var(--star)" opacity=".35"/>');
  g.innerHTML = out.join("");

  // Blink through five frames: two of them bad, tagged and renamed the way the filters do it.
  var demo = document.getElementById("bf-demo"), field = document.getElementById("bf-field");
  var name = document.getElementById("bf-demo-name"), tag = document.getElementById("bf-demo-tag"), count = document.getElementById("bf-demo-count");
  var frames = [
    { n: 41 }, { n: 42, bad: "trail" }, { n: 43 }, { n: 44, bad: "blur" }, { n: 45 }
  ];
  function show(f) {
    var id = "000" + f.n;
    name.textContent = "frame_" + id + (f.bad ? "_" + f.bad : "") + ".fits";
    count.textContent = f.n + " / 218";
    tag.textContent = f.bad || "";
    demo.classList.toggle("bad", !!f.bad);
    demo.classList.toggle("is-trail", f.bad === "trail");
    if (f.bad === "blur") field.setAttribute("filter", "url(#bf-soft)");
    else field.removeAttribute("filter");
  }
  // With reduced motion, hold the trail frame instead of blinking.
  var still = window.matchMedia && window.matchMedia("(prefers-reduced-motion: reduce)").matches;
  if (still) { show(frames[1]); return; }
  var at = 0;
  show(frames[at]);
  setInterval(function () {
    at = (at + 1) % frames.length;
    show(frames[at]);
  }, 900);
})();
</script>
