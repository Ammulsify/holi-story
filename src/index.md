---
theme: air
---
```js
import * as d3 from "npm:d3";
const web = await FileAttachment("data/all_holi_web.csv").csv({typed: true});
const mariah = await FileAttachment("data/mariah_web.csv").csv({typed: true});
```

```js
const songs = [
  {key: "Rang Barse Silsila", label: "Rang Barse"},
  {key: "Hori Khele Raghuveera", label: "Holi Khele Raghuveera"},
  {key: "BALAM PICHKARI", label: "Balam Pichkari"},
  {key: "Do Me A Favor", label: "Do Me A Favour"},
  {key: "Holiyaan", label: "Holiyaan"}
];

const colorMap = {
  "Rang Barse": "#e63946",
  "Holi Khele Raghuveera": "#f4a261",
  "Balam Pichkari": "#2a9d8f",
  "Do Me A Favour": "#457b9d",
  "Holiyaan": "#9b5de5"
};

const opacityMap = {
  "Rang Barse": 1,
  "Holi Khele Raghuveera": 0.8,
  "Balam Pichkari": 0.8,
  "Do Me A Favour": 0.6,
  "Holiyaan": 0.6
};

const long = web.flatMap(row => {
  const date = new Date(row.Time);
  return songs.map(s => ({
    date,
    year: date.getFullYear(),
    month: date.getMonth() + 1,
    song: s.label,
    interest: +row[s.key] || 0
  }));
}).sort((a, b) => a.date - b.date);

const mariahAll = mariah.map(row => ({
  date: new Date(row.Time),
  interest: +row["All I Want for Christmas Is You"] || 0
})).sort((a, b) => a.date - b.date);

const rangBarseAll = long
  .filter(d => d.song === "Rang Barse")
  .sort((a, b) => a.date - b.date);
```

```js
function drawHeartbeat(data, color, label) {
  const width = 360;
  const height = 220;
  const margin = {top: 30, right: 20, bottom: 35, left: 40};

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .style("background", "#ffffff")
    .style("border-radius", "4px");

  const x = d3.scaleTime()
    .domain(d3.extent(data, d => d.date))
    .range([margin.left, width - margin.right]);

  const y = d3.scaleLinear()
    .domain([0, d3.max(data, d => d.interest)])
    .range([height - margin.bottom, margin.top]);

  const area = d3.area()
    .x(d => x(d.date))
    .y0(height - margin.bottom)
    .y1(d => y(d.interest))
    .curve(d3.curveMonotoneX);

  svg.append("path")
    .datum(data)
    .attr("fill", color)
    .attr("opacity", 0.07)
    .attr("d", area);

  const line = d3.line()
    .x(d => x(d.date))
    .y(d => y(d.interest))
    .curve(d3.curveMonotoneX);

  const path = svg.append("path")
    .datum(data)
    .attr("fill", "none")
    .attr("stroke", color)
    .attr("stroke-width", 1.8)
    .attr("d", line);

  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x).ticks(5).tickSize(0))
    .call(g => g.selectAll("text").style("fill", "#aaa").style("font-size", "10px").attr("dy", "1.2em"))
    .call(g => g.select(".domain").attr("stroke", "#e5e5e5"))
    .call(g => g.selectAll(".tick line").remove());

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).ticks(4).tickSize(-width + margin.left + margin.right))
    .call(g => g.selectAll("text").style("fill", "#aaa").style("font-size", "10px"))
    .call(g => g.select(".domain").remove())
    .call(g => g.selectAll(".tick line").attr("stroke", "#f0f0f0").attr("stroke-dasharray", "2,2"));

  svg.append("text")
    .attr("x", margin.left)
    .attr("y", margin.top - 10)
    .attr("text-anchor", "start")
    .attr("fill", color)
    .attr("font-size", "11px")
    .attr("font-weight", "700")
    .attr("letter-spacing", "0.05em")
    .text(label.toUpperCase());

  return svg.node();
}
```

```js
function drawSlideChart(visibleSongs) {
  const width = 480;
  const height = 340;
  const margin = {top: 30, right: 30, bottom: 45, left: 45};

  const filtered = long.filter(d => visibleSongs.includes(d.song));
  const focusSong = visibleSongs[visibleSongs.length - 1];

  const marchValues = filtered.filter(d => d.month === 3).map(d => d.interest).sort(d3.ascending);
  const p90 = d3.quantile(marchValues, 0.90);
  const absoluteMax = d3.max(filtered, d => d.interest);
  const maxY = Math.min(absoluteMax, (p90 || absoluteMax) * 2.5);

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .style("background", "#ffffff")
    .style("overflow", "visible");

  const x = d3.scaleTime()
    .domain(d3.extent(long, d => d.date))
    .range([margin.left, width - margin.right]);

  const y = d3.scaleLinear()
    .domain([0, maxY])
    .range([height - margin.bottom, margin.top]);

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).ticks(5).tickSize(-(width - margin.left - margin.right)))
    .call(g => g.select(".domain").remove())
    .call(g => g.selectAll(".tick line").attr("stroke", "#f5f5f5").attr("stroke-dasharray", "3,3"))
    .call(g => g.selectAll(".tick text").remove());

  const line = d3.line()
    .x(d => x(d.date))
    .y(d => y(d.interest))
    .curve(d3.curveMonotoneX);

  const bySong = d3.group(filtered, d => d.song);
  const songOrder = ["Holiyaan", "Do Me A Favour", "Balam Pichkari", "Holi Khele Raghuveera", "Rang Barse"];

  songOrder.forEach(song => {
    const values = bySong.get(song);
    if (!values) return;

    const isFocus = song === focusSong;
    const isRangBarse = song === "Rang Barse";
    const baseOpacity = isFocus || isRangBarse ? 1 : 0.12;

    const area = d3.area()
      .x(d => x(d.date))
      .y0(height - margin.bottom)
      .y1(d => y(Math.min(d.interest, maxY)))
      .curve(d3.curveMonotoneX);

    svg.append("path")
      .datum(values)
      .attr("class", `area song-area song-${song.replace(/\s+/g, "-")}`)
      .attr("fill", colorMap[song])
      .attr("opacity", (isFocus || isRangBarse) ? 0.07 : 0.02)
      .attr("d", area);

    const clippedValues = values.map(d => ({...d, interest: Math.min(d.interest, maxY)}));

    svg.append("path")
      .datum(clippedValues)
      .attr("class", `line song-line song-${song.replace(/\s+/g, "-")}`)
      .attr("fill", "none")
      .attr("stroke", colorMap[song])
      .attr("stroke-width", isFocus ? 2.8 : isRangBarse ? 2.2 : 1.2)
      .attr("opacity", baseOpacity)
      .attr("d", line);
  });

  const keyMoments = {
    "Rang Barse":            { date: new Date("2019-03-01"), label: "Peak: 2019" },
    "Holi Khele Raghuveera": { date: new Date("2019-03-01"), label: "Holi Khele peaks: 2019" },
    "Balam Pichkari":        { date: new Date("2013-03-01"), label: "↑ Peaked at 100 in 2013" },
    "Holiyaan":              { date: new Date("2024-03-01"), label: "Holiyaan 2024 — barely registers" }
  };

  const km = keyMoments[focusSong];
  if (km) {
    const songData = long.filter(d => d.song === focusSong);
    const match = songData.find(d =>
      d.date.getFullYear() === km.date.getFullYear() &&
      d.date.getMonth() === km.date.getMonth()
    );
    if (match) {
      const cx = x(match.date);
      const cy = y(Math.min(match.interest, maxY));

      svg.append("circle")
        .attr("cx", cx).attr("cy", cy).attr("r", 4.5)
        .attr("fill", colorMap[focusSong])
        .attr("stroke", "#fff")
        .attr("stroke-width", 2);

      const boxW = km.label.length * 6.5 + 20;
      const boxH = 26;
      const boxX = cx > width * 0.55 ? cx - boxW - 8 : cx + 10;
      const boxY = cy - boxH - 6;

      svg.append("rect")
        .attr("x", boxX).attr("y", boxY)
        .attr("width", boxW).attr("height", boxH)
        .attr("rx", 4)
        .attr("fill", colorMap[focusSong])
        .attr("opacity", 0.92);

      svg.append("text")
        .attr("x", boxX + boxW / 2)
        .attr("y", boxY + boxH / 2 + 4)
        .attr("text-anchor", "middle")
        .attr("font-size", "9.5px")
        .attr("font-family", "Georgia, serif")
        .attr("font-weight", "700")
        .attr("fill", "#fff")
        .text(km.label);
    }
  }

  let labelOffset = 0;
  visibleSongs.forEach(song => {
    const isFocus = song === focusSong;
    const isRangBarse = song === "Rang Barse";
    if (!isFocus && !isRangBarse) return;

    svg.append("text")
      .attr("x", margin.left + 6)
      .attr("y", margin.top + labelOffset)
      .attr("font-size", "9px")
      .attr("font-weight", "700")
      .attr("font-family", "Georgia, serif")
      .attr("fill", colorMap[song])
      .attr("opacity", 0.85)
      .text(song);

    labelOffset += 13;
  });

  const tooltip = d3.select("body").selectAll(".slide-tooltip").data([1]).join("div")
    .attr("class", "slide-tooltip")
    .style("position", "fixed")
    .style("background", "#111")
    .style("color", "#fff")
    .style("padding", "10px 14px")
    .style("border-radius", "6px")
    .style("font-size", "11px")
    .style("font-family", "Georgia, serif")
    .style("pointer-events", "none")
    .style("opacity", 0)
    .style("z-index", 1000)
    .style("line-height", "1.9")
    .style("box-shadow", "0 4px 24px rgba(0,0,0,0.2)")
    .style("min-width", "170px");

  const bisect = d3.bisector(d => d.date).left;

  const cursorLine = svg.append("line")
    .attr("stroke", "#e0e0e0")
    .attr("stroke-width", 1)
    .attr("stroke-dasharray", "4,3")
    .attr("y1", margin.top)
    .attr("y2", height - margin.bottom)
    .attr("opacity", 0);

  svg.append("rect")
    .attr("x", margin.left).attr("y", margin.top)
    .attr("width", width - margin.left - margin.right)
    .attr("height", height - margin.top - margin.bottom)
    .attr("fill", "none")
    .attr("pointer-events", "all")
    .on("mousemove", function(event) {
      const [mx] = d3.pointer(event, this);
      const xPos = mx + margin.left;
      const xDate = x.invert(xPos);

      cursorLine.attr("x1", xPos).attr("x2", xPos).attr("opacity", 1);

      svg.selectAll(".song-line").attr("opacity", 0.04);
      svg.selectAll(".song-area").attr("opacity", 0.01);

      let closestSong = null;
      let closestDist = Infinity;
      visibleSongs.forEach(song => {
        const songData = long.filter(d => d.song === song);
        const i = bisect(songData, xDate, 1);
        const d = songData[Math.min(i, songData.length - 1)];
        if (d) {
          const screenY = y(Math.min(d.interest, maxY));
          const dist = Math.abs(screenY - event.offsetY);
          if (dist < closestDist) { closestDist = dist; closestSong = song; }
        }
      });

      if (closestSong) {
        svg.selectAll(`.song-line.song-${closestSong.replace(/\s+/g, "-")}`).attr("opacity", 1);
        svg.selectAll(`.song-area.song-${closestSong.replace(/\s+/g, "-")}`).attr("opacity", 0.1);
      }

      const lines = [];
      visibleSongs.forEach(song => {
        const songData = long.filter(d => d.song === song);
        const i = bisect(songData, xDate, 1);
        const d = songData[Math.min(i, songData.length - 1)];
        if (d) {
          const isHighlight = song === closestSong;
          lines.push(`<span style="color:${colorMap[song]};font-weight:${isHighlight ? '700' : '400'};opacity:${isHighlight ? '1' : '0.4'}">${song}: ${d.interest}</span>`);
        }
      });

      tooltip
        .style("opacity", 1)
        .style("left", (event.clientX + 18) + "px")
        .style("top", (event.clientY - 50) + "px")
        .html(`<div style="color:#f9c74f;font-weight:700;font-size:12px;margin-bottom:6px">${xDate.getFullYear()}</div>${lines.join("<br>")}`);
    })
    .on("mouseleave", () => {
      tooltip.style("opacity", 0);
      cursorLine.attr("opacity", 0);
      songOrder.forEach(song => {
        const isFocus = song === focusSong;
        const isRangBarse = song === "Rang Barse";
        const op = isFocus || isRangBarse ? 1 : 0.12;
        svg.selectAll(`.song-line.song-${song.replace(/\s+/g, "-")}`).attr("opacity", op);
        svg.selectAll(`.song-area.song-${song.replace(/\s+/g, "-")}`).attr("opacity", (isFocus || isRangBarse) ? 0.07 : 0.02);
      });
    });

  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x).ticks(5).tickSize(0))
    .call(g => g.selectAll("text").style("fill", "#bbb").style("font-size", "10px").attr("dy", "1.4em"))
    .call(g => g.select(".domain").attr("stroke", "#eee"))
    .call(g => g.selectAll(".tick line").remove());

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).ticks(4).tickSize(0))
    .call(g => g.selectAll("text").style("fill", "#bbb").style("font-size", "10px"))
    .call(g => g.select(".domain").remove());

  svg.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -(height / 2))
    .attr("y", 12)
    .attr("text-anchor", "middle")
    .attr("font-size", "9px")
    .attr("fill", "#ddd")
    .attr("letter-spacing", "0.08em")
    .text("SEARCH INTEREST");

  return svg.node();
}
```

```html
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }

  body {
    background: #ffffff;
    color: #111;
    font-family: 'Georgia', serif;
    overflow-x: hidden;
  }

  #observablehq-main,
  #observablehq-main > div {
    max-width: 100% !important;
    padding: 0 !important;
    margin: 0 !important;
    width: 100% !important;
  }

  .section-1 {
    height: 100vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    text-align: center;
    padding: 2rem;
    background: #ffffff;
  }

  .section-1 .byline {
    font-size: 0.8rem;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    color: #bbb;
    margin-bottom: 1.5rem;
  }

  .section-1 h1 {
    font-size: clamp(2rem, 6vw, 4.5rem);
    font-weight: 700;
    line-height: 1.2;
    color: #111;
    max-width: 860px;
    margin-bottom: 1.5rem;
  }

  .section-1 h1 .gradient-text {
    background: linear-gradient(90deg, #e63946, #f4a261, #f9c74f, #43aa8b, #9b5de5);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .scroll-hint {
    margin-top: 3rem;
    font-size: 0.75rem;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    color: #ccc;
    animation: bounce 2s infinite;
  }

  .scroll-hint::after { content: " ↓"; }

  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(6px); }
  }

  .section-2 {
    min-height: 100vh;
    background: #ffffff;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 6rem 2rem;
  }

  .section-2-inner {
    max-width: 860px;
    width: 100%;
    text-align: center;
    margin: 0 auto;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .section-label {
    font-size: 0.7rem;
    letter-spacing: 0.25em;
    text-transform: uppercase;
    color: #bbb;
    margin-bottom: 1.2rem;
  }

  .section-2-title {
    font-size: clamp(1.8rem, 4vw, 3rem);
    font-weight: 700;
    color: #111;
    line-height: 1.25;
    margin-bottom: 0.5rem;
    text-align: center;
    width: 100%;
  }

  .section-2-title .gradient-text {
    background: linear-gradient(90deg, #e63946, #f4a261, #f9c74f, #43aa8b, #9b5de5);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .section-2-subtitle {
    font-size: 1rem;
    color: #888;
    max-width: 560px;
    margin: 1rem auto 3rem;
    line-height: 1.8;
  }

  .two-charts {
    display: flex;
    flex-direction: row;
    gap: 3rem;
    justify-content: center;
    align-items: flex-start;
    margin-top: 3rem;
  }

  .chart-block {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 360px;
  }

  .chart-block svg {
    width: 100%;
    height: auto;
  }

  .chart-sublabel {
    font-size: 0.72rem;
    color: #bbb;
    margin-top: 0.8rem;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    text-align: center;
  }

  .section-2-caption {
    font-size: 0.72rem;
    color: #ccc;
    margin-top: 2rem;
    letter-spacing: 0.05em;
  }

  .section-3-wrapper {
    width: 100%;
    height: 100vh;
    overflow: hidden;
  }

  .section-3-inner {
    display: flex;
    width: 400%;
    height: 100%;
  }

  .slide {
    width: 25%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #ffffff;
    padding: 4rem 5rem;
    gap: 5rem;
    border-right: 1px solid #f0f0f0;
  }

  .slide-chart {
    flex-shrink: 0;
    width: 480px;
  }

  .slide-text {
    max-width: 300px;
  }

  .slide-number {
    font-size: 0.7rem;
    letter-spacing: 0.25em;
    text-transform: uppercase;
    color: #ccc;
    margin-bottom: 1.2rem;
  }

  .slide-title {
    font-size: clamp(1.4rem, 2.5vw, 2rem);
    color: #111;
    line-height: 1.3;
    margin-bottom: 1rem;
    font-weight: 700;
  }

  .slide-title .gradient-text {
    background: linear-gradient(90deg, #e63946, #f4a261, #f9c74f, #43aa8b, #9b5de5);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .slide-body {
    font-size: 0.9rem;
    color: #888;
    line-height: 1.8;
  }

  .section-4 {
    background: #ffffff;
  }

  .s4-card {
    height: 100vh;
    width: 100%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    text-align: center;
    padding: 4rem 2rem;
    max-width: 100%;
    position: sticky;
    top: 0;
    opacity: 0;
    transform: translateY(50px);
    background: #ffffff;
    z-index: 1;
  }

  .s4-label {
    font-size: 0.7rem;
    letter-spacing: 0.25em;
    text-transform: uppercase;
    color: #ccc;
    margin-bottom: 2.5rem;
  }

  .s4-statement {
    font-size: clamp(1.6rem, 4vw, 3.2rem);
    font-weight: 700;
    color: #111;
    line-height: 1.3;
    margin-bottom: 0.8rem;
  }

  .s4-statement .gradient-text {
    background: linear-gradient(90deg, #e63946, #f4a261, #f9c74f, #43aa8b, #9b5de5);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .s4-quote {
    font-size: clamp(1.1rem, 2.5vw, 1.7rem);
    font-style: italic;
    color: #444;
    line-height: 1.7;
    max-width: 680px;
    border-left: 3px solid #e63946;
    padding-left: 1.8rem;
    text-align: left;
    margin-bottom: 1rem;
  }

  .s4-source {
    font-size: 0.8rem;
    color: #bbb;
    text-align: left;
    padding-left: 1.8rem;
    letter-spacing: 0.05em;
  }

  /* ── SECTION 5 — METHODOLOGY ── */
.section-5 {
  background: #ffffff;
  padding: 6rem 2rem 8rem;
  border-top: 1px solid #f0f0f0;
}

.section-5-inner {
  max-width: 680px;
  margin: 0 auto;
}

.s5-scroll-hint {
  font-size: 0.8rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: #ccc;
  text-align: center;
  margin-bottom: 4rem;
}

.s5-divider {
  width: 40px;
  height: 2px;
  background: linear-gradient(90deg, #e63946, #9b5de5);
  margin: 0 auto 4rem;
}

.s5-heading {
  font-size: 0.75rem;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  color: #bbb;
  margin-bottom: 0.8rem;
  margin-top: 2.5rem;
}

.s5-body {
  font-size: 0.9rem;
  color: #888;
  line-height: 1.9;
  margin-bottom: 1rem;
}

.s5-songs {
  margin-top: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.s5-song {
  font-size: 0.9rem;
  color: #666;
  display: flex;
  align-items: center;
  gap: 0.6rem;
  line-height: 1.6;
}

.s5-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}

.s5-credit {
  margin-top: 3rem;
  font-size: 0.75rem;
  color: #ccc;
  letter-spacing: 0.08em;
  text-align: center;
}

/* ── RESPONSIVE / MOBILE ── */
@media (max-width: 768px) {

  /* Section 1 */
  .section-1 h1 {
    font-size: clamp(1.8rem, 8vw, 3rem);
  }

  /* Section 2 */
  .section-2-inner {
    padding: 0 1rem;
  }
  .two-charts {
    flex-direction: column;
    gap: 1.5rem;
  }
  .two-charts svg {
    width: 100% !important;
    height: auto !important;
  }
  .chart-wrapper {
    width: 100%;
  }

  /* Section 3 — disable horizontal scroll, go vertical */
  .section-3-wrapper {
    height: auto !important;
    overflow: visible !important;
  }
  .section-3-inner {
    display: flex !important;
    flex-direction: column !important;
    width: 100% !important;
    height: auto !important;
  }
  .slide {
    width: 100% !important;
    height: auto !important;
    flex-direction: column !important;
    padding: 3rem 1.5rem !important;
    gap: 2rem !important;
    min-height: 100vh;
  }
  .slide-chart {
    width: 100% !important;
    order: 2;
  }
  .slide-chart svg {
    width: 100% !important;
    height: auto !important;
  }
  .slide-text {
    order: 1;
    max-width: 100% !important;
  }

  /* Section 4 */
  .s4-card {
    padding: 2rem 1.5rem !important;
  }
  .s4-card h2 {
    font-size: clamp(1.4rem, 6vw, 2.5rem) !important;
  }

  /* Section 5 */
  .section-5 {
    padding: 4rem 1.5rem 6rem !important;
  }
}
</style>
```

<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/gsap.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/ScrollTrigger.min.js"></script>

```html
<!-- SECTION 1 -->
<section class="section-1">
  <p class="byline">By Nirukti Nayak</p>
  <h1>India's Holi Playlist.<br>Frozen for <span class="gradient-text">40 Years.</span></h1>
  <p class="scroll-hint">Scroll to explore</p>
</section>

<!-- SECTION 2 -->
<section class="section-2">
  <div class="section-2-inner">
    <p class="section-label">A Global Phenomenon</p>
    <h2 class="section-2-title">Music is how we <span class="gradient-text">Decorate Time.</span></h2>
    <p class="section-2-subtitle">Some songs beccome the festival. When Mariah Carey recorded All I Want For Christmas Is You in four days in 1994. She did not know she was writing a law. Rang Barse came out in 1981, a film song, disposable by design. It outlasted the film, the decade, the century. The spike arrives every year not because anyone decided to play these songs, but because nobody decided not to. That is the difference between a hit and a deeply rooted cultural memory.</p>
    <div class="two-charts" id="two-charts"></div>
    <p class="section-2-caption">Google Trends search interest · Left: Worldwide · Right: India</p>
  </div>
</section>

<!-- SECTION 3 -->
<div class="section-3-wrapper" id="js-wrapper">
  <div class="section-3-inner" id="js-slideContainer">

    <div class="slide">
      <div class="slide-chart" id="chart-slide-1"></div>
      <div class="slide-text">
        <p class="slide-number">01 / 04</p>
        <h2 class="slide-title">Every March,<br><span class="gradient-text">without fail.</span></h2>
        <p class="slide-body">Since 2004, one song spikes every Holi. A 1981 track called Rang Barse, so fused with the festival that searching for it has become a ritual in itself.</p>
      </div>
    </div>

    <div class="slide">
      <div class="slide-chart" id="chart-slide-2"></div>
      <div class="slide-text">
        <p class="slide-number">02 / 04</p>
        <h2 class="slide-title">It's not<br><span class="gradient-text">alone.</span></h2>
        <p class="slide-body">Holi Khele Raghuveera follows the same pattern. Another classic, another annual ritual. The Holi playlist has more than one song but not many that dethrones Rang Barse.</p>
      </div>
    </div>

    <div class="slide">
      <div class="slide-chart" id="chart-slide-3"></div>
      <div class="slide-text">
        <p class="slide-number">03 / 04</p>
        <h2 class="slide-title">One song tried<br><span class="gradient-text">to break in.</span></h2>
        <p class="slide-body">Balam Pichkari from Yeh Jawaani Hai Deewani exploded in 2013. For a few years it looked like a new Holi anthem was born. Then it started fading. The door was closing again.</p>
      </div>
    </div>

    <div class="slide">
      <div class="slide-chart" id="chart-slide-4"></div>
      <div class="slide-text">
        <p class="slide-number">04 / 04</p>
        <h2 class="slide-title">The playlist<br><span class="gradient-text">is frozen.</span></h2>
        <p class="slide-body">Holiyaan released in 2024 with John Abraham and a full Bollywood budget. On this chart it is invisible. A 1981 song still beats everything new. India already decided what Holi sounds like.</p>
      </div>
    </div>

  </div>
</div>

<!-- SECTION 4 -->
<section class="section-4">

  <div class="s4-card" id="s4-1">
    <p class="s4-label">The Last Challenger</p>
    <h2 class="s4-statement">Balam Pichkari was released in 2013.</h2>
    <h2 class="s4-statement">It was the last song to even come close.</h2>
    <h2 class="s4-statement"><span class="gradient-text">That was 13 years ago.</span></h2>
  </div>

  <div class="s4-card" id="s4-2">
    <p class="s4-label">Since Then</p>
    <h2 class="s4-statement">Bollywood has released hundreds of Holi songs.</h2>
    <h2 class="s4-statement"><span class="gradient-text">None of them registered.</span></h2>
  </div>

  <div class="s4-card" id="s4-3">
    <p class="s4-label">The Internet Agrees</p>
    <blockquote class="s4-quote">"We've got AI, we've got multiverse movies... but we haven't had a new Holi anthem since 2013 after Balam Pichkari."</blockquote>
    <p class="s4-source">— r/BollywoodMusic</p>
  </div>

  <div class="s4-card" id="s4-4">
    <p class="s4-label">The Conclusion</p>
    <h2 class="s4-statement">India already decided what Holi sounds like.</h2>
    <h2 class="s4-statement"><span class="gradient-text">It decided a long time ago.</span></h2>
  </div>

</section>

<section class="section-5">
  <div class="section-5-inner">

    <div class="s5-scroll-hint">
      ↑ Scroll back up to experience the story
    </div>

    <div class="s5-divider"></div>

    <h3 class="s5-heading">About this story</h3>
    <p class="s5-body">This piece explores Google Trends search interest data for five Bollywood Holi songs from 2004 to 2025, alongside Mariah Carey's All I Want For Christmas Is You as a global reference point for how festivals fossilize music.</p>

    <h3 class="s5-heading">Data & Methodology</h3>
    <p class="s5-body">Search interest data was collected from Google Trends in March 2026. Data begins in 2004 because that is the earliest date for which Google Trends provides reliable monthly data. All values are normalised on a scale of 0–100, where 100 represents the peak search interest for that query over the selected period.</p>
    <p class="s5-body">Songs were selected based on being explicitly Holi-themed Bollywood tracks released across different decades representing a mix of enduring classics and new challengers. The comparison query was run as a combined search to preserve relative normalisation across all five songs.</p>
    <p class="s5-body">Mariah Carey data was collected as a separate worldwide query and is used for narrative framing only it is not directly comparable to the India-specific Holi data.</p>

    <h3 class="s5-heading">Songs in this story</h3>
    <div class="s5-songs">
      <div class="s5-song"><span class="s5-dot" style="background:#e63946"></span><strong>Rang Barse</strong> — Silsila (1981)</div>
      <div class="s5-song"><span class="s5-dot" style="background:#f4a261"></span><strong>Holi Khele Raghuveera</strong> — Baghban (2003)</div>
      <div class="s5-song"><span class="s5-dot" style="background:#2a9d8f"></span><strong>Balam Pichkari</strong> — Yeh Jawaani Hai Deewani (2013)</div>
      <div class="s5-song"><span class="s5-dot" style="background:#9b5de5"></span><strong>Holiyaan</strong> — Dream Girl 2 (2024)</div>
    </div>

    <p class="s5-credit">Reported and designed by Nirukti Nayak · March 2026</p>

  </div>
</section>
```

```js
```js
setTimeout(() => {
  const container = document.getElementById("two-charts");
  if (!container || container.children.length > 0) return;

  const b1 = document.createElement("div");
  b1.className = "chart-block";
  b1.appendChild(drawHeartbeat(mariahAll, "#9b5de5", "All I Want For Christmas Is You"));
  const sl1 = document.createElement("p");
  sl1.className = "chart-sublabel";
  sl1.textContent = "Spikes every December · Worldwide";
  b1.appendChild(sl1);

  const b2 = document.createElement("div");
  b2.className = "chart-block";
  b2.appendChild(drawHeartbeat(rangBarseAll, "#e63946", "Rang Barse"));
  const sl2 = document.createElement("p");
  sl2.className = "chart-sublabel";
  sl2.textContent = "Spikes every March · India";
  b2.appendChild(sl2);

  container.appendChild(b1);
  container.appendChild(b2);
}, 600);
```
```js
setTimeout(() => {
  const slideConfigs = [
    ["Rang Barse"],
    ["Rang Barse", "Holi Khele Raghuveera"],
    ["Rang Barse", "Holi Khele Raghuveera", "Balam Pichkari"],
    ["Rang Barse", "Holi Khele Raghuveera", "Balam Pichkari", "Holiyaan"]
  ];

  slideConfigs.forEach((songs, i) => {
    const el = document.getElementById(`chart-slide-${i + 1}`);
    if (el) {
      el.innerHTML = "";
      el.appendChild(drawSlideChart(songs));
    }
  });
}, 600);
```
```js
gsap.registerPlugin(ScrollTrigger);

setTimeout(() => {

  // Horizontal scroll — desktop only
  if (window.innerWidth > 768) {
    const wrapper = document.querySelector("#js-wrapper");
    const container = document.querySelector("#js-slideContainer");

    if (wrapper && container) {
      const totalScroll = container.scrollWidth - window.innerWidth;
      gsap.to(container, {
        x: -totalScroll,
        ease: "none",
        scrollTrigger: {
          trigger: wrapper,
          pin: true,
          scrub: 1,
          start: "top top",
          end: "+=" + totalScroll
        }
      });
    }
  }

  // Section 4 cards — always
  const s4cards = document.querySelectorAll(".s4-card");

  s4cards.forEach((card, i) => {
    ScrollTrigger.create({
      trigger: card,
      start: "top 75%",
      onEnter: () => gsap.to(card, { opacity: 1, y: 0, duration: 0.8, ease: "power3.out" }),
      onLeaveBack: () => gsap.to(card, { opacity: 0, y: 50, duration: 0.5, ease: "power2.in" })
    });

    if (i < s4cards.length - 1) {
      ScrollTrigger.create({
        trigger: card,
        start: "bottom 25%",
        onEnter: () => gsap.to(card, { opacity: 0, y: -60, scale: 0.95, duration: 0.5, ease: "power2.in" }),
        onLeaveBack: () => gsap.to(card, { opacity: 1, y: 0, scale: 1, duration: 0.6, ease: "power3.out" })
      });
    }
  });

  // Section 2 fade in — always
  gsap.fromTo(".section-2-inner",
    { opacity: 0, y: 40 },
    {
      opacity: 1, y: 0, duration: 1, ease: "power2.out",
      scrollTrigger: {
        trigger: ".section-2",
        start: "top 70%",
        toggleActions: "play none none reverse"
      }
    }
  );

}, 800);
```