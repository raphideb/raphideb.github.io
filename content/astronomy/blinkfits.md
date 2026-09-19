---
title: "BlinkFits"
linkTitle: "BlinkFits"
description: "A free, lightweight Windows blink comparator for FITS subs: spot asteroids, novae, satellite trails and bad frames."
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
  position: absolute; left: 0; right: 0; top: 0; display: flex; justify-content: space-between;
  padding: 8px 12px; font-size: .78rem; color: #9098A0; background: rgba(10, 11, 13, .75);
}
.bf .frame-bar .name { color: #E6E8EB; }
.bf .blink-caption { font-size: .88rem; color: var(--dim); margin: 12px 0 0; }
.bf .blink-caption b { color: var(--mark); font-weight: 600; }
.bf .state-b .obj-a, .bf .state-a .obj-b { opacity: 0; }
.bf .ghost { opacity: 0; }
.bf .show-ghost .ghost { opacity: 1; }

.bf .shot { margin: 0 0 40px; }
.bf .shot img { display: block; width: 100%; height: auto; border: 1px solid var(--border); border-radius: 10px; }
.bf .shot figcaption { font-size: .88rem; color: var(--dim); margin-top: 12px; }

.bf .grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 16px; margin-bottom: 40px; }
.bf .card { background: var(--panel); border: 1px solid var(--border); border-radius: 10px; padding: 20px; }
.bf .card h3 { margin: 0 0 6px; font-size: 1.02rem; display: flex; align-items: center; gap: 10px; }
.bf .card h3::before { content: ""; width: 8px; height: 8px; border-radius: 50%; background: var(--accent); flex: none; }
.bf .card.hl h3::before { background: var(--mark); }
.bf .card p { margin: 0; color: var(--dim); font-size: .95rem; }

.bf .two { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 40px; margin-bottom: 40px; }
.bf .two table { width: 100%; font-size: .95rem; }
.bf .two td:first-child { white-space: nowrap; padding-right: 20px; }

.bf .note { background: var(--panel); border: 1px solid var(--border); border-left: 3px solid var(--mark); border-radius: 8px; padding: 18px 20px; margin-bottom: 40px; }
.bf .note p { margin: 0 0 10px; color: var(--dim); }
.bf .note p:last-child { margin: 0; }
.bf .note strong { color: var(--text); }

.bf .credits { border-top: 1px solid var(--border); padding-top: 24px; font-size: .88rem; color: var(--dim); }
.bf .credits p { margin: 0 0 6px; }
</style>

<div class="bf">

<div class="bf-hero">
<div>
<div class="bf-brand"><img src="/astronomy/blinkfits-icon.png" alt="" width="32" height="32"><span>Blink<span class="fits">Fits</span> <span class="ver">1.0</span></span></div>
<p class="eyebrow">Blink comparator for FITS</p>
<p class="headline">Blink your subs.<br>Spot what moved.</p>
<p class="lead">BlinkFits flips back and forth between frames, with zoom, pan and stretch held perfectly still. Asteroids, novae, variable stars, satellite trails and subs worth throwing away jump out at you.</p>
<div class="bf-download">
<a class="btn-dl" href="https://github.com/raphideb/blinkfits_release/releases/latest/download/blinkfits.zip">
<svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M12 3v12"/><path d="m6 10 6 6 6-6"/><path d="M4 20h16"/></svg>
<span>Download latest version for Windows<small>blinkfits.zip</small></span>
</a>
<p class="hint">Need an older version? <a href="https://github.com/raphideb/blinkfits_release/releases">All releases are on GitHub</a>.</p>
<p class="hint">Extract the zip, then run <code>blinkfits.exe</code>. No installer needed.</p>
<div class="sha">
<div class="sha-label"><span>Check it in PowerShell, in the extracted folder</span><span id="bf-copied" aria-live="polite"></span></div>
<div class="sha-box"><code id="bf-check">(Get-FileHash .\blinkfits.exe).Hash -eq (Get-Content .\blinkfits.exe.sha256).Split()[0]</code><button class="copy" id="bf-copy" type="button">Copy</button></div>
<p class="hint"><code>True</code> means the exe matches the SHA-256 in <code>blinkfits.exe.sha256</code>, which comes in the zip.</p>
<p class="hint">Windows warns when you start it? <a href="#download-help">Here is what to do</a>.</p>
</div>
<ul class="facts">
<li>Free</li>
<li>Windows 10 / 11, 64 bit</li>
<li>No installer</li>
<li>Offline, no telemetry</li>
</ul>
</div>
</div>
<div class="blink" role="img" aria-label="Animated example: two frames of a star field blinked against each other, one object moves">
<div class="frame" id="bf-demo">
<svg viewBox="0 0 640 400" preserveAspectRatio="xMidYMid slice" aria-hidden="true">
<g id="bf-stars"></g>
<circle class="ghost" cx="262" cy="222" r="7" fill="none" stroke="var(--sky-mark)" stroke-width="1.6" opacity=".6"/>
<circle class="obj-a" cx="262" cy="222" r="3.2" fill="var(--star)"/>
<circle class="obj-b" cx="318" cy="236" r="3.2" fill="var(--star)"/>
</svg>
<div class="frame-bar mono"><span class="name" id="bf-demo-name">frame_00041.fits</span><span id="bf-demo-count">41 / 218</span></div>
</div>
<p class="blink-caption">Every star stays put. The one that jumps is your <b>minor planet</b>.</p>
</div>
</div>

<figure class="shot">
<img src="/astronomy/blinkfits-screenshot.jpg" width="1600" height="1157" loading="lazy" alt="BlinkFits with 210 raw frames of NGC 7331 loaded: file list with Move marked to folder on the left, the debayered frame in the middle, stretch, zoom, auto blink, marking suffix and language settings on the right, seven frames marked with the _bad suffix">
<figcaption>210 raw subs of NGC 7331, debayered from the header's Bayer pattern and shown with the asinh stretch. Marked frames are renamed with the _bad suffix and shown in orange.</figcaption>
</figure>

## Made for a night's worth of subs {#features}

<p class="sub">Load a folder, press the arrow keys, and see what changed. Everything else stays out of the way.</p>

<div class="grid">
<div class="card hl">
<h3>The view never moves</h3>
<p>Zoom, pan and stretch belong to the viewer, not the frame. Switch images and only the sky changes, which is the whole point of blinking.</p>
</div>
<div class="card">
<h3>Six stretches</h3>
<p>Auto (STF) by default, plus levels, asinh, logarithmic, histogram equalisation and unstretched. Sliders react instantly, even on large frames.</p>
</div>
<div class="card">
<h3>Auto blink</h3>
<p>From 0.1 s to 60 s per frame, wrapping at both ends. Space or Pause stops and resumes at the same speed, Reverse runs it backwards.</p>
</div>
<div class="card hl">
<h3>Mark the keepers (or the rejects)</h3>
<p>One key renames the file on disk with your suffix: <code>frame.fits</code> becomes <code>frame_tag.fits</code>. It still shows next session, and in Explorer.</p>
</div>
<div class="card">
<h3>One shot colour</h3>
<p>Debayer raw OSC frames from <code>BAYERPAT</code> in the header, or pick RGGB, BGGR, GRBG or GBRG. Each channel is stretched on its own for a neutral sky.</p>
</div>
<div class="card">
<h3>FITS header</h3>
<p>Show the header cards in place of the image. Keep blinking and the header follows, frame by frame: exposure, gain, temperature, time.</p>
</div>
<div class="card">
<h3>Several folders</h3>
<p>Add files and folders from different sessions to one list, drop any of them again, or pass paths on the command line.</p>
</div>
<div class="card">
<h3>Careful with memory</h3>
<p>16 bit samples, a cache that fits your RAM, and a clear message instead of swapping your machine to a halt.</p>
</div>
<div class="card">
<h3>15 languages</h3>
<p>English, Deutsch, Français, Italiano, Español, Português (BR / PT), Nederlands, Dansk, Svenska, Polski, Русский, Українська, 日本語 and 中文.</p>
</div>
</div>

<div class="two">
<div>

## Hands on the keyboard {#keys}

<p class="sub">Blinking is fastest without the mouse.</p>

| Key | Action |
|-----|--------|
| <kbd>←</kbd> <kbd>→</kbd> &nbsp;or&nbsp; <kbd>A</kbd> <kbd>D</kbd> | Previous / next image; during auto blink, turn it round |
| <kbd>Space</kbd> | Start / stop auto blink |
| <kbd>Home</kbd> <kbd>End</kbd> | First / last image |
| <kbd>M</kbd> | Mark or unmark the current image |
| <kbd>S</kbd> | Next stretch method |
| <kbd>1</kbd> … <kbd>6</kbd> | Zoom 5, 10, 25, 50, 75, 100 % |
| <kbd>F</kbd> or <kbd>0</kbd> | Fit to window |
| <kbd>N</kbd> | Night vision: the interface in dim red, the image unchanged |
| <kbd>F1</kbd> | Help |
| <kbd>Esc</kbd> | Close the FITS header, Help or About |

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
- Settings live in `%APPDATA%\blinkfits`. Delete the exe and the folder and it is gone.

</div>
</div>

## Windows warns when you start BlinkFits? {#download-help}

<p class="sub">BlinkFits is a hobby project and the exe is not code signed. Windows treats new, unsigned programs with caution until enough people have run them.</p>

<div class="note">
<p><strong>Check the file.</strong> The zip contains <code>blinkfits.exe.sha256</code> with the SHA-256 of the exe. After extracting, run the PowerShell check above, or run <code>certutil -hashfile blinkfits.exe SHA256</code> in a command prompt and compare the result with that file. If they match, the exe arrived intact.</p>
<p><strong>Windows SmartScreen</strong> (“Windows protected your PC”): click <em>More info</em>, then <em>Run anyway</em>.</p>
</div>

<div class="two">
<div>

## Terms of use {#terms}

<p class="sub">BlinkFits may be downloaded and used free of charge by anyone, for any purpose. Redistributing, selling or modifying it, or passing it off as your own work, is not permitted without the author's written permission. It is provided as is, without any warranty.</p>

</div>
<div>

## Contact {#contact}

<p class="sub">Questions, bug reports, a FITS file that will not open? Write to <a href="mailto:raphi@crashdump.ch">raphi@crashdump.ch</a>. Please share a link to this page rather than the download, so people always get the current version and its checksum.</p>
<p class="sub"><a href="https://paypal.me/RaphaelDebinski" target="_blank" rel="noopener">Support me with PayPal</a></p>

</div>
</div>

<div class="credits">
<p>© 2026 Raphael Debinski. All rights reserved.</p>
<p>Built with Go, Gio and go-text/typesetting; their open source licenses are shown in the program under About and apply to those components only.</p>
</div>

</div>

<script>
(function () {
  // Copy the checksum command.
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

  // Blink between two frames. With reduced motion, show both positions instead.
  var demo = document.getElementById("bf-demo"), name = document.getElementById("bf-demo-name"), count = document.getElementById("bf-demo-count");
  var still = window.matchMedia && window.matchMedia("(prefers-reduced-motion: reduce)").matches;
  if (still) {
    demo.classList.add("show-ghost", "state-b");
    return;
  }
  var b = false;
  demo.classList.add("state-a");
  setInterval(function () {
    b = !b;
    demo.classList.toggle("state-a", !b);
    demo.classList.toggle("state-b", b);
    demo.classList.toggle("show-ghost", b);
    name.textContent = b ? "frame_00042.fits" : "frame_00041.fits";
    count.textContent = b ? "42 / 218" : "41 / 218";
  }, 700);
})();
</script>
