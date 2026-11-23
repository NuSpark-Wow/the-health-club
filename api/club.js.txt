// The Health Club — server function (Vercel) — Learn...Wow! / NuSparkLearnWow
// One endpoint, several modes:
//   exercise — how to do an exercise (steps, breathing, mistakes, easier/harder, sample ranges), source-checked
//   machine  — how to set up and use a gym machine, source-checked
//   reallife — Train for Real Life: muscles and moves behind an everyday task
//   home     — The Home Gym: moves that use the equipment the visitor has at home
//   muscle   — Body Map: what a muscle does and exercises that work it
//   diagram  — draws a start/end position diagram (SVG) for an exercise, stretch or machine
//   label    — Label Reader: explains a photo of a nutrition label in plain English
//   topic    — Nutrition Corner and Lifestyle Lounge explainers, source-checked
//   myth     — a set of Myth or Fact cards, source-checked
//   debate   — The Great Debate: Coach Mike, Coach Lisa and Dr. Data debate a question, with proof charts
//   ask      — follow-up questions to one guide or the whole panel
//   plan     — printable sample ideas (routine, 30-day challenge, desk stretches, food swaps)
// Honesty rule: every source shown is a page Claude actually retrieved with web search during THIS request.
// Education rule: The Health Club educates and presents ideas. It is not a doctor, dietitian or personal trainer.

const MODEL = process.env.ANTHROPIC_MODEL || "claude-sonnet-5-5";
const FAST_MODEL = process.env.FAST_MODEL || "claude-haiku-5-5";
// Diagrams use the fast model so they draw quickly. To trade speed for detail, set DIAGRAM_MODEL to claude-sonnet-5-5 in Vercel.
const DIAGRAM_MODEL = process.env.DIAGRAM_MODEL || FAST_MODEL;
const DAILY_LIMIT = parseInt(process.env.DAILY_LIVE_LIMIT || "60", 10);
const ALLOWED = (process.env.ALLOWED_ORIGINS || "").split(",").map(s => s.trim()).filter(Boolean);

const GUIDES = {
  mike: "Coach Mike, the fitness educator: exercises, gym machines, strength, cardio and the Body Map. Upbeat, direct and motivating. He teaches and explains; he is not anyone's personal trainer.",
  lisa: "Coach Lisa, the healthy-living educator: stretching, mobility, food and drinks, sleep, stress and habits. Warm, calm and practical. She teaches and explains; she is not a dietitian.",
  data: "Dr. Data, the skeptical researcher who joins The Great Debate: she brings the studies, separates strong evidence from weak, and says plainly when the science is mixed. A friendly character, not a medical doctor."
};
const GUIDE_IDS = Object.keys(GUIDES);

const ZONES = {
  kids: "KIDS' PLAYGROUND (ages about 5-12). Playful, simple words, short sentences, fun moves (animal walks, jumping games, balance games). Say to do it with a grown-up nearby. ABSOLUTELY NO weight, weight loss, calories, dieting, body size or 'burning fat' content. Food talk is only fun facts, 'eat the rainbow', water and trying new foods.",
  adults: "ADULTS. Clear, friendly, practical. General ranges are fine; never personal targets.",
  fifty: "50+ CORNER. Joint-friendly, balance and fall prevention in mind. Always show the easier or seated version first, suggest holding a chair or wall for balance where it helps, and give a clear 'check with your doctor before starting something new' reminder.",
  family: "FAMILY WORKOUT. Parents and kids together; moves everyone can do side by side, with a kid version and a grown-up version. Kid rules apply: NO weight, calorie, dieting or body-size content."
};
const zoneOf = z => (ZONES[z] ? z : "adults");
const isKidZone = z => z === "kids" || z === "family";

// --- simple per-IP daily limit (resets when the server instance restarts; the Anthropic spend cap is the real backstop)
const hits = new Map();
function overLimit(ip, limit) {
  const key = ip + "|" + new Date().toISOString().slice(0, 10);
  const n = (hits.get(key) || 0) + 1;
  hits.set(key, n);
  if (hits.size > 5000) hits.clear();
  return n > limit;
}

const SYSTEM = `You are the editor of "The Health Club", an interactive healthy-living learning tool from NuSparkLearnWow.com (part of the Learn...Wow! series from NuSpark Publishing). The host is Mr. Paul, a warm, upbeat teacher. Two friendly guides teach in the club: Coach Mike (exercise, machines, strength) and Coach Lisa (stretching, food and drinks, sleep, stress, habits). Dr. Data, a skeptical researcher, joins The Great Debate.

WHAT THE CLUB IS: an education tool. It explains, shows and presents ideas about exercise, stretching, gym machines, food and drinks, eating styles and healthy habits. It is NOT a doctor, dietitian, or personal trainer, and never acts like one.

EDUCATION RULES (always):
1. Educate, don't prescribe. Explain and suggest ("many people...", "one idea is...", "a common starting point is..."). Never say "your plan", "you must" or "you should eat/do X". Printables are always "sample ideas".
2. General ranges only, e.g. "many beginners start with 2-3 sets of 8-12 reps". NEVER give calorie targets, macro or gram targets, weight-loss amounts, target weights or BMI judgments tied to the person.
3. Doctor nudge: if the visitor mentions pain, injury, surgery, pregnancy, medication, a medical condition, dizziness, chest pain, shortness of breath or an eating concern, say clearly to check with a doctor first and keep the answer general and gentle. Never diagnose or suggest treatment.
4. Emergencies (chest pain, fainting, trouble breathing, a severe allergic reaction, thoughts of self-harm): the first line tells them to stop and call 911 (or local emergency services) right away. For self-harm or crisis, also mention the 988 Suicide & Crisis Lifeline (call or text 988 in the US).
5. Never comment on anyone's body, weight or looks. No "fat burning", "toning problem areas" or "bikini body" language. Focus on strength, energy, movement, sleep and feeling good.
6. Eating styles (plant-based, vegan, keto, Mediterranean, gluten-free, intermittent fasting and others) are explained NEUTRALLY with fair pros and cons and what the research says. Never recommend one over another.
7. Gluten, celiac disease, allergies and intolerances: explain them, but a diagnosis always comes from a doctor, and never suggest cutting out food groups to self-treat or self-diagnose.
8. Never promote extreme or unsafe practices: crash diets, very low calorie eating, fasting for kids or teens, purging, laxatives, diet pills, supplements for weight loss, steroids, dehydration, exercising through pain. If asked, explain kindly why it is risky and suggest talking with a doctor.
9. Supplements: general education only (what it is, what the research says), always "ask a doctor or pharmacist before taking any supplement".

HONESTY RULES:
- Use the web_search tool when it is available. Base answers on reliable sources: government health agencies (CDC, NIH, MedlinePlus, FDA, USDA, MyPlate), major medical centers (Mayo Clinic, Cleveland Clinic, Johns Hopkins, Harvard Health), professional bodies (American College of Sports Medicine, American Council on Exercise, American Heart Association, Academy of Nutrition and Dietetics, Celiac Disease Foundation) and peer-reviewed research. Avoid forums, influencer sites, supplement sellers and social media.
- Every source you list MUST be a URL that appeared in your search results in this conversation. Never invent or guess URLs. Never invent statistics or study results.
- If the science is mixed or unsure, say so plainly.

TOPIC RULES:
- Stay on exercise, stretching, gym machines, food and drinks, nutrition education, eating styles, sleep, stress, posture and healthy habits. If a question is off-topic, set ok to false and kindly suggest The Answer Man on NuSparkLearnWow.com.
- No brand recommendations, no product rankings, no "near me" answers. Product TYPES are fine (for example "a yoga mat", "plain Greek yogurt").
- National, everyday examples. US measurements, with metric in parentheses where it helps.

WRITING STYLE: clear, friendly and encouraging, like a great teacher at a health club. Short sentences, plain words; explain any term the first time. Never call the tool or its content "free." No markdown, no asterisks, plain sentences.

OUTPUT: Never put citation tags or markup (like <cite ...>) inside the JSON text. Reply with ONLY a JSON object wrapped in <json> and </json> tags, in the exact shape the user message asks for.`;

const FOOTER_NOTE = "For education and ideas only. Not a substitute for a doctor, dietitian or certified trainer.";
const sourcesShape = `"sources": [{"title": "Publisher: page title", "url": "https://..."}]`;
const S = (v, n) => String(v == null ? "" : v).slice(0, n);

// ---------- shared helpers ----------
function normUrl(u) {
  try { const x = new URL(String(u).trim()); return (x.hostname.replace(/^www\./, "") + x.pathname.replace(/\/+$/, "")).toLowerCase(); }
  catch (e) { return ""; }
}
function hostOf(u) { try { return new URL(String(u).trim()).hostname.replace(/^www\./, "").toLowerCase(); } catch (e) { return ""; } }
const words = t => new Set(String(t || "").toLowerCase().split(/[^a-z0-9]+/).filter(w => w.length > 3));

// strong-source filter: government, medical, professional and established health publications only
const STRONG_SUFFIX = [".edu", ".gov", ".mil", ".int", ".ac.uk", ".gov.uk", ".nhs.uk", ".gc.ca", ".edu.au", ".gov.au"];
const STRONG_HOSTS = [
  "mayoclinic.org", "clevelandclinic.org", "my.clevelandclinic.org", "hopkinsmedicine.org", "mountsinai.org", "uclahealth.org", "stanfordhealthcare.org", "mgh.harvard.edu", "kp.org", "intermountainhealthcare.org", "nebraskamed.com", "uofmhealth.org",
  "nhs.uk", "who.int", "heart.org", "stroke.org", "diabetes.org", "arthritis.org", "cancer.org", "lung.org", "kidney.org", "aarp.org", "ncoa.org", "sleepfoundation.org", "aasm.org", "apa.org", "nami.org",
  "acsm.org", "acefitness.org", "nasm.org", "nsca.com", "exrx.net", "physio-pedia.com", "orthoinfo.aaos.org", "aaos.org", "apta.org", "choosept.com", "spinehealth.com", "spine-health.com",
  "eatright.org", "celiac.org", "beyondceliac.org", "gluten.org", "foodallergy.org", "aaaai.org", "acaai.org", "kidshealth.org", "healthychildren.org", "aap.org", "nutrition.org", "oldwayspt.org", "wholegrainscouncil.org", "foodinsight.org", "cspi.org",
  "medlineplus.gov", "nih.gov", "cdc.gov", "fda.gov", "usda.gov", "myplate.gov", "nutrition.gov", "health.gov", "hhs.gov", "nia.nih.gov", "niddk.nih.gov", "ods.od.nih.gov", "pubmed.ncbi.nlm.nih.gov", "ncbi.nlm.nih.gov",
  "health.harvard.edu", "hsph.harvard.edu", "nutritionsource.hsph.harvard.edu", "bmj.com", "thelancet.com", "jamanetwork.com", "nejm.org", "nature.com", "sciencedirect.com", "frontiersin.org", "springer.com", "wiley.com", "plos.org", "cochrane.org", "cochranelibrary.com", "examine.com",
  "webmd.com", "verywellfit.com", "verywellhealth.com", "healthline.com", "medicalnewstoday.com", "everydayhealth.com", "health.com", "prevention.com", "consumerreports.org",
  "britannica.com", "wikipedia.org", "en.wikipedia.org", "nytimes.com", "npr.org", "bbc.com", "bbc.co.uk", "scientificamerican.com", "smithsonianmag.com", "reuters.com", "apnews.com", "washingtonpost.com", "theatlantic.com", "pbs.org"
];
function isStrong(url) {
  const h = hostOf(url); if (!h) return false;
  if (STRONG_SUFFIX.some(sfx => h.endsWith(sfx))) return true;
  return STRONG_HOSTS.some(d => h === d || h.endsWith("." + d));
}
function verifySources(list, retrieved) {
  const out = [], seen = new Set(), pages = [...retrieved.values()];
  for (const s of Array.isArray(list) ? list : []) {
    if (!s || !s.url) continue;
    let r = retrieved.get(normUrl(s.url));
    if (!r) {
      const host = hostOf(s.url), want = words((s.title || "") + " " + s.url);
      let best = null, bestScore = -1;
      for (const pg of pages) {
        if (hostOf(pg.url) !== host) continue;
        const have = words(pg.title + " " + pg.url);
        let score = 0; want.forEach(w => { if (have.has(w)) score++; });
        if (score > bestScore) { best = pg; bestScore = score; }
      }
      r = best;
    }
    if (!r || !isStrong(r.url) || /[?&]comment/i.test(r.url)) continue;
    const n = normUrl(r.url);
    if (seen.has(n)) continue;
    seen.add(n);
    out.push({ title: String(r.title || s.title || r.url).slice(0, 200), url: r.url });
  }
  return out.slice(0, 6);
}
function extractJson(text) {
  let raw = null;
  const low = text.toLowerCase();
  for (let i = low.lastIndexOf("<json>"); i >= 0; i = low.lastIndexOf("<json>", i - 1)) {
    const end = low.indexOf("</json>", i);
    if (end > i) { raw = text.slice(i + 6, end); break; }
    if (i === 0) break;
  }
  if (!raw) { const a = text.indexOf("{"), z = text.lastIndexOf("}"); if (a >= 0 && z > a) raw = text.slice(a, z + 1); }
  if (!raw) return null;
  raw = raw.trim().replace(/^```(?:json)?/i, "").replace(/```$/, "").trim();
  try { return JSON.parse(raw); } catch (e) { return null; }
}
const cleanText = s => String(s || "").replace(/<\/?cite\b[^>]*>/gi, "").replace(/<\/?[a-z_]+:[a-z_]+\b[^>]*>/gi, "").replace(/\*\*/g, "").replace(/[ \t]{2,}/g, " ").trim();
const clip = (s, n) => cleanText(s).slice(0, n);
const clipList = (a, n, len) => (Array.isArray(a) ? a : []).slice(0, n).map(x => clip(x, len)).filter(Boolean);
const oneOf = (v, list, d) => (list.includes(v) ? v : d);
const cleanSteps = (a, n) => (Array.isArray(a) ? a : []).slice(0, n).map(s => ({ title: clip(s && s.title, 120), text: clip(s && s.text, 1400), check: clip(s && s.check, 400) })).filter(s => s.text);

// Diagram safety: keep only the SVG, strip anything active (it is also shown as an <img>, which never runs scripts)
function cleanSvg(text) {
  let s = String(text || "");
  const a = s.indexOf("<svg");
  if (a < 0) return "";
  let z = s.lastIndexOf("</svg>");
  if (z < a) { // cut off before the end: drop the unfinished last tag and close the drawing
    const lastClose = s.lastIndexOf(">");
    if (lastClose <= a) return "";
    s = s.slice(0, lastClose + 1).replace(/<(g|text|tspan)\b[^>]*>(?![\s\S]*<\/\1>)/gi, "") + "</svg>";
    z = s.lastIndexOf("</svg>");
  }
  s = s.slice(a, z + 6)
    .replace(/<script[\s\S]*?<\/script>/gi, "")
    .replace(/<foreignObject[\s\S]*?<\/foreignObject>/gi, "")
    .replace(/<image\b[^>]*>/gi, "")
    .replace(/<a\b[^>]*>|<\/a>/gi, "")
    .replace(/\son[a-z]+\s*=\s*("[^"]*"|'[^']*')/gi, "")
    .replace(/\s(xlink:)?href\s*=\s*("[^"]*"|'[^']*')/gi, "")
    .replace(/@import[^;]*;?/gi, "")
    .replace(/url\(\s*['"]?(?!#)[^)]*\)/gi, "none");
  if (!/xmlns=/.test(s.slice(0, 300))) s = s.replace("<svg", '<svg xmlns="http://www.w3.org/2000/svg"');
  return s.length > 90000 ? "" : s;
}

async function anthropic(messages, maxSearches, model, maxTokens) {
  const body = { model: model || MODEL, max_tokens: maxTokens || 9000, system: SYSTEM, messages };
  // the fast model must have thinking turned off, or it spends its whole budget thinking and returns nothing (the stronger model refuses this setting)
  if (/haiku/i.test(body.model)) body.thinking = { type: "disabled" };
  if (maxSearches > 0) body.tools = [{ type: "web_search_20250305", name: "web_search", max_uses: maxSearches }];
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "x-api-key": process.env.ANTHROPIC_API_KEY, "anthropic-version": "2023-06-01", "content-type": "application/json" },
    body: JSON.stringify(body)
  });
  if (!r.ok) { const t = await r.text(); throw new Error("Anthropic API error " + r.status + ": " + t.slice(0, 300)); }
  return r.json();
}
const textOf = data => (data.content || []).filter(c => c.type === "text").map(c => c.text).join("");

async function callClaude(userPrompt, maxSearches) {
  const messages = [{ role: "user", content: userPrompt }];
  const retrieved = new Map();
  let finalText = "";
  const absorb = data => {
    for (const block of data.content || []) {
      if (block.type === "web_search_tool_result" && Array.isArray(block.content)) {
        for (const res of block.content) if (res && res.url) retrieved.set(normUrl(res.url), { url: res.url, title: res.title || "" });
      }
      if (block.type === "text") {
        finalText += block.text;
        for (const c of block.citations || []) if (c && c.url) retrieved.set(normUrl(c.url), { url: c.url, title: c.title || "" });
      }
    }
  };
  let data = null;
  for (let round = 0; round < 4; round++) {
    data = await anthropic(messages, maxSearches);
    absorb(data);
    if (data.stop_reason === "pause_turn") { messages.push({ role: "assistant", content: data.content }); continue; }
    break;
  }
  if (!extractJson(finalText) && data) {
    messages.push({ role: "assistant", content: data.content });
    messages.push({ role: "user", content: "Thank you. Now reply with ONLY the final <json>...</json> object in the exact shape requested, using only the pages you already found. Do not search again." });
    try { const d2 = await anthropic(messages, 1); absorb(d2); } catch (e) { console.error(e); }
  }
  return { text: finalText, retrieved };
}


// ---------- prompts ----------
function zoneLine(b) { return `AGE ZONE: ${ZONES[zoneOf(b.zone)]}`; }

function modePrompt(mode, b) {
  const zl = zoneLine(b);

  if (mode === "exercise" || mode === "machine") {
    const name = S(b.name, 140);
    if (!name) return null;
    const isMachine = mode === "machine";
    return `${zl}
The visitor wants to learn ${isMachine ? "how to set up and use this gym machine" : "how to do this exercise or stretch"}: "${name}".
${isMachine ? "" : "If it is really a gym machine, still explain it as an exercise done on that machine."}
Research it with web search, then teach it well. This is a learning page, so make it rich: about double the length of a quick answer, but every sentence useful.
Shape:
{"ok": true,
 "title": "proper name of the ${isMachine ? "machine" : "exercise"}",
 "what": "2-3 sentences: what it is and why people do it",
 "muscles": ["main muscles worked, plain names with the science name in parentheses where helpful"],
 "equipment": "what you need, or 'No equipment'",
 ${isMachine ? `"setup": ["3-6 short steps to adjust the seat, pads, handles and weight pin before starting"],` : ""}
 "steps": [{"title": "short step name", "text": "1-3 clear sentences"}],
 "breathing": "one or two sentences on when to breathe in and out",
 "mistakes": ["4-6 common mistakes, each with the fix in the same sentence"],
 "easier": [{"name": "easier version", "how": "one sentence"}],
 "harder": [{"name": "harder version", "how": "one sentence"}],
 "sample": "one sentence with a GENERAL starting range many people use (sets, reps or time, and rest). Kid zones: playful counts or seconds, no weights.",
 "safety": "one or two sentences: when to stop and who should check with a doctor first",
 "coachTip": "one memorable tip from Coach Mike",
 "deeper": [{"heading": "a deeper-learning heading, e.g. Why form matters, How it helps everyday life", "text": "3-5 sentences"}],
 "followups": ["3 short follow-up questions a visitor might ask next"],
 ${sourcesShape}}
Give 4-8 steps, 1-3 easier and 1-3 harder versions, 2-3 deeper sections. If the request is off-topic or unsafe, reply {"ok": false, "message": "kind one-sentence explanation"}.`;
  }

  if (mode === "reallife") {
    const goal = S(b.goal, 160);
    if (!goal) return null;
    const names = (Array.isArray(b.muscles) ? b.muscles : []).slice(0, 20).map(x => S(x, 40)).join(", ");
    return `${zl}
TRAIN FOR REAL LIFE. The visitor's everyday goal: "${goal}".
Work backward from the task: which muscles and abilities (strength, balance, mobility, endurance) it needs, and which exercises build toward it, from easiest to hardest. This is education and sample ideas, not a personal program. Use web search to check the facts, and find one genuinely surprising, true "Wow...Who Knew?" fact about this task or the body doing it (from a source you retrieved).
For "muscles", use ONLY names from this exact list so they light up on the body map: ${names}.
Shape:
{"ok": true,
 "title": "the goal, cleaned up",
 "why": "2-3 sentences: what this task asks of the body",
 "muscles": [{"muscle": "a name from the list", "why": "one sentence on its job in this task"}],
 "moves": [{"name": "exercise name", "level": "Easier" | "Medium" | "Harder", "where": "Home" | "Gym" | "Anywhere", "how": "one sentence"}],
 "practice": "1-2 sentences on safely practicing the real task itself (e.g. practice getting up from the floor next to a sturdy couch)",
 "wow": "one surprising, true fact, 1-2 sentences",
 "coachTip": "one tip from Coach Mike",
 "safety": "one or two sentences: who should check with a doctor and when to stop",
 ${sourcesShape}}
Give 3-5 muscles and 4-6 moves ordered easiest to hardest, mostly doable at home. If off-topic or unsafe, reply {"ok": false, "message": "kind one-sentence explanation"}.`;
  }

  if (mode === "home") {
    const eq = (Array.isArray(b.equipment) ? b.equipment : []).slice(0, 14).map(x => S(x, 40)).filter(Boolean);
    return `${zl}
THE HOME GYM. The visitor exercises at home. Equipment they have: ${eq.length ? eq.join(", ") : "NOTHING (bodyweight only)"}.
${b.small ? "SMALL SPACE: every move must fit in about a 6 by 6 foot (2 by 2 meter) area, like a living room or apartment.\n" : ""}${b.quiet ? "QUIET: no jumping, stomping or dropping weights (apartment-friendly).\n" : ""}Suggest moves that use ONLY what they have (bodyweight moves are always fine). Use the equipment creatively and safely: a sturdy non-rolling chair for step-ups, dips, incline push-ups, seated moves and balance; a wall for wall sits and wall push-ups; a towel for sliding or stretching; water jugs as light weights.
Shape:
{"ok": true,
 "intro": "2 sentences on what they can do with this setup",
 "setup": ["2-4 short tips to set up the space safely for this equipment"],
 "moves": [{"name": "exercise name (include the equipment, e.g. 'Chair step-up', 'Band row')", "equipment": "what it uses, or 'No equipment'", "area": "Upper body" | "Lower body" | "Core" | "Cardio" | "Stretch & mobility" | "Balance", "how": "one sentence"}],
 "circuit": {"name": "a short name", "rounds": "e.g. 'Do 2-3 rounds, resting 1 minute between rounds'", "items": [{"name": "move name from the list", "amount": "general range, e.g. 10-12 reps or 30 seconds"}]},
 "coachTip": "one tip from Coach Mike about working out at home",
 "safety": "one or two sentences on home-workout safety"}
Give 10-14 moves covering several areas (kid zones: playful moves only, no weights; 50+: balance and seated options first) and a 5-6 move sample circuit.`;
  }

  if (mode === "muscle") {
    const m = S(b.muscle, 80);
    if (!m) return null;
    return `${zl}
Body Map: the visitor tapped "${m}".
Shape:
{"ok": true,
 "muscle": "plain name (science name)",
 "job": "2-3 sentences: what this muscle group does",
 "everyday": "one sentence: where you use it in everyday life",
 "exercises": [{"name": "exercise name", "equipment": "None / Dumbbells / Machine / Band", "how": "one sentence"}],
 "order": "one or two sentences: where these fit in a workout (big muscles before small, and so on)",
 "stretch": {"name": "a simple stretch for this area", "how": "one or two sentences"},
 "tip": "one tip from Coach Mike"}
Give 5-6 exercises from easiest to hardest, mixing no-equipment, dumbbell and machine options (kid zones: no machines or weights, playful moves only).`;
  }

  if (mode === "diagram") {
    const name = S(b.name, 140);
    if (!name) return null;
    const kind = b.kind === "machine" ? "gym machine exercise" : b.kind === "stretch" ? "stretch" : "exercise";
    const kidNote = isKidZone(zoneOf(b.zone)) ? "Draw the figure as a friendly kid-like figure." : "";
    return `Draw a teaching diagram for this ${kind}: "${name}". ${S(b.hint, 400)}
Rules for the drawing:
- One clean, simple SVG, like an illustration in a good fitness book. Flat colors, clear outlines, plenty of white space. ${kidNote}
- <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 460">. A white background rect first.
- TWO PANELS side by side: left panel labeled "1. START", right panel labeled "2. FINISH" (for a held stretch: "1. GET SET" and "2. HOLD"). A thin light-gray divider between them.
- In each panel, a simple, well-proportioned human figure (rounded head, torso, arms and legs drawn as thick rounded strokes or capsules, in slate #3d4a5c). The pose must be physically correct for the movement. Draw the floor line, and any bench, ball, dumbbell, band or machine simply.
- Highlight the main working muscles on the figure with semi-transparent coral #ff6f59 shapes.
- Add 2-4 curved arrows in teal #0f8f7a showing the direction of movement.
- Use font-family="Arial, Helvetica, sans-serif" for all text, at least 15px. Keep every label inside the canvas and never let text overlap the figures.
- A short title at the top center (the exercise name), 23px bold, color #1f2a44. Under each panel's label leave the figure on a light gray floor line (#cfd6de) at about y=368.
- Make the figure look like a real person in a clean infographic style: a solid slate (#3d4a5c) silhouette with a round head, a shaped torso (wider chest, narrower waist) and tapered limbs (thicker at the top), joints smoothly connected; draw the far-side arm and leg in a lighter gray-blue (#8d99ab) behind the body.
- At the bottom, add two centered lines: "Works: <main muscles>" in bold coral #c94a37 at y=406, and 2-3 short form cues joined like "✓ Cue one   ✓ Cue two" in bold dark teal #0b5a4c at y=436.
- Keep it compact: about 45 shapes or fewer, no gradients, no filters, no images, no scripts, no links, no foreignObject, no external fonts, no CSS animations.
- Be accurate. If unsure, keep it simpler rather than wrong.
Reply with exactly two things and nothing else:
CAPTION: one sentence describing what the diagram shows
then the complete <svg ...>...</svg> code.`;
  }

  if (mode === "label") {
    return `${zl}
Label Reader: the image is a photo of a food or drink package. Read the Nutrition Facts panel and ingredient list if visible, and explain it in plain English. EXPLAIN, DON'T JUDGE: never call a food good or bad, never tell the visitor to eat or avoid it. Describe what the numbers and words mean.
Shape:
{"ok": true,
 "product": "what the product appears to be (generic, e.g. 'a breakfast cereal'), no brand needed",
 "servingSize": "the serving size as printed, and what that looks like in real life",
 "servings": "servings per container, and what that means if you eat the whole package",
 "items": [{"label": "line from the panel, e.g. Total Sugars", "amount": "amount as printed with %DV if shown", "meaning": "one plain-English sentence"}],
 "sugarTsp": "added sugars (or total sugars if added is not shown) converted to teaspoons, about 4 grams per teaspoon, e.g. 'About 3 teaspoons of added sugar per serving'; empty string if not readable",
 "dvNote": "one sentence on how to read % Daily Value (5% or less is low, 20% or more is high)",
 "ingredients": ["2-5 plain-English notes on the ingredient list: what's listed first, words that mean sugar, any of the major allergens shown in a Contains statement"],
 "terms": [{"term": "a label word like 'enriched' or 'whole grain'", "meaning": "one sentence"}],
 "bigPicture": "2-3 neutral sentences tying it together",
 "kidFact": "one fun, simple fact for kids about this kind of food (no calories or weight talk)"}
If parts are blurry, say what you could and could not read. If the image is not a food or drink label, reply {"ok": false, "message": "kind sentence asking for a clear photo of the Nutrition Facts label"}.${isKidZone(zoneOf(b.zone)) ? " KID ZONE: keep it very simple, skip calories and %DV numbers in the explanations, focus on fun facts and what the words mean." : ""}`;
  }

  if (mode === "topic") {
    const room = b.room === "lifestyle" ? "Lifestyle Lounge (Coach Lisa: sleep, stress, screen time, posture, habits)" : b.room === "homegym" ? "The Home Gym (Coach Mike: exercising at home and simple home equipment)" : "Nutrition Corner (Coach Lisa: food for health)";
    const station = S(b.station, 60);
    const topic = S(b.topic, 200);
    if (!topic) return null;
    const stationRules = {
      choose: "Station: HOW TO CHOOSE. Teach how to choose this food or aisle item at the store: what to look for on the shelf and label, what the common words mean, marketing words to ignore, and simple swap ideas.",
      styles: "Station: EATING STYLES EXPLAINED. Explain this eating style NEUTRALLY: what it is, typical foods, why people choose it, what the research says (strong, mixed or weak), things to watch for, and common myths. Never recommend it or argue against it. Remind that anyone with a health condition should talk with a doctor or registered dietitian before changing how they eat.",
      sensitivities: "Station: GLUTEN & SENSITIVITIES. Explain clearly and accurately. Distinguish medical conditions (like celiac disease or a food allergy) from intolerances and lifestyle choices. A diagnosis always comes from a doctor; testing for celiac should happen BEFORE cutting out gluten. Never suggest self-diagnosis or cutting food groups to self-treat. Mention that severe allergic reactions are an emergency (call 911).",
      equipment: "Station: EQUIPMENT 101. Explain how to choose, set up and safely use this kind of simple home exercise equipment: what to look for, sizes or strengths, common mistakes, and safety checks. Product TYPES only, no brands or stores.",
      drinks: "Station: DRINKS. Explain what is in this drink type, how it fits into a day, and general facts people often get wrong (caffeine, sugar, hydration). General ranges from official sources only.",
      lifestyle: "Explain this healthy-habit topic with practical, everyday ideas people can try, backed by reliable sources."
    };
    return `${zl}
Room: ${room}. ${stationRules[station] || stationRules.lifestyle}
The visitor picked or typed: "${topic}".
Research with web search, then teach it. A rich learning page, about double the length of a quick answer.
Shape:
{"ok": true,
 "title": "clear title",
 "intro": "2-3 sentence overview",
 "sections": [{"heading": "short heading", "text": "3-5 sentences"}],
 "tryThis": ["3-5 simple ideas people can try (ideas, not orders)"],
 "myths": [{"myth": "a common myth", "truth": "what's actually true, one or two sentences"}],
 "research": "1-3 sentences: what the research says and how strong it is",
 "doctorNote": "one sentence on when to talk with a doctor or registered dietitian (empty string if not needed)",
 "followups": ["3 short follow-up questions"],
 ${sourcesShape}}
Give 3-5 sections and 1-3 myths. If off-topic, reply {"ok": false, "message": "kind one-sentence explanation"}.`;
  }

  if (mode === "myth") {
    const cat = { exercise: "exercise and fitness", food: "food, drinks and nutrition", lifestyle: "sleep, stress and everyday habits" }[b.category] || "a mix of exercise, food and everyday health habits";
    const avoid = (Array.isArray(b.avoid) ? b.avoid : []).slice(0, 24).map(x => "- " + S(x, 120)).join("\n");
    return `${zl}
Myth or Fact Wall: create 6 cards about ${cat}. Mix them: about half myths, the rest facts or "partly true". Use statements people really hear.
${avoid ? "Do NOT repeat these statements:\n" + avoid : ""}
Check each one with web search.
Shape:
{"cards": [{"statement": "the statement as people say it", "answer": "myth" | "fact" | "partly", "explain": "2-3 sentences on what the research says", "source": {"title": "Publisher: page title", "url": "https://..."}}]}`;
  }

  if (mode === "debate") {
    const q = S(b.question, 220);
    if (!q) return null;
    return `${zl}
THE GREAT DEBATE. Question from the visitor: "${q}"
Panel: Coach Mike (id "mike"), Coach Lisa (id "lisa") and Dr. Data (id "data"). Give Mike and Lisa different, reasonable positions that real experts hold (not strawmen). Dr. Data focuses on what the evidence does and doesn't show. All three stay friendly and respectful, and the whole debate is education, not personal advice.
Research with web search FIRST. Proof charts must use numbers that appear on pages you actually retrieved, with that page's URL. If you cannot find real numbers, give fewer charts or none. Never invent numbers.
Shape:
{"ok": true,
 "question": "the question, cleaned up",
 "positions": {"mike": "Mike's position in under 12 words", "lisa": "Lisa's position in under 12 words", "data": "Dr. Data's angle in under 12 words"},
 "opening": [{"id": "mike", "text": "about 60 words"}, {"id": "lisa", "text": "about 60 words"}, {"id": "data", "text": "about 60 words"}],
 "rounds": [[{"id": "mike", "text": "rebuttal, 40-60 words"}, {"id": "lisa", "text": "..."}, {"id": "data", "text": "..."}], [{"id": "lisa", "text": "..."}, {"id": "mike", "text": "..."}, {"id": "data", "text": "..."}]],
 "charts": [{"title": "short chart title", "unit": "unit, e.g. % lower risk or minutes per week", "bars": [{"label": "short label", "value": 12.5}], "note": "one sentence explaining the chart", "sourceTitle": "Publisher: page title", "sourceUrl": "https://..."}],
 "wrap": {"best": [{"id": "mike", "point": "each side's strongest point, one sentence"}, {"id": "lisa", "point": "..."}], "research": "Dr. Data's 'what the research actually says', 3-4 sentences", "bottomLine": "one balanced closing sentence"},
 "vote": ["short label for Mike's side", "short label for Lisa's side", "It depends"],
 ${sourcesShape}}
Give 0-2 charts with 2-6 bars each. If the question is off-topic or unsafe, reply {"ok": false, "message": "kind one-sentence explanation"}.`;
  }

  if (mode === "ask") {
    const to = GUIDES[b.to] ? b.to : "all";
    const ctx = S(b.context, 2500);
    const hist = (Array.isArray(b.history) ? b.history : []).slice(-6).map(h => `${h.who === "user" ? "Visitor" : "Guide"}: ${S(h.text, 500)}`).join("\n");
    const q = S(b.question, 600);
    if (!q) return null;
    const panel = b.panel === "debate" ? "mike, lisa and data" : "mike and lisa";
    return `${zl}
${ctx ? "What the visitor is looking at:\n" + ctx + "\n" : ""}${hist ? "Conversation so far:\n" + hist + "\n" : ""}The visitor asks ${to === "all" ? "the guides (" + panel + ")" : GUIDES[to].split(",")[0]}: "${q}"
Guides:
${GUIDE_IDS.map(id => "- " + id + ": " + GUIDES[id]).join("\n")}
Answer as ${to === "all" ? "one or two of " + panel + " (whoever fits best; two only if they add different angles)" : "only " + to}. Use web search if facts are needed.
If the question is too vague to answer well, ask ONE short clarifying question in "clarify" and keep the reply brief.
Shape:
{"replies": [{"id": "mike" | "lisa" | "data", "text": "a helpful answer, 60-180 words"}], "clarify": "one short clarifying question or empty string", ${sourcesShape}}`;
  }

  if (mode === "plan") {
    const kind = S(b.kind, 20);
    const a = b.answers && typeof b.answers === "object" ? b.answers : {};
    const ans = Object.entries(a).slice(0, 6).map(([k, v]) => `- ${S(k, 40)}: ${S(Array.isArray(v) ? v.join(", ") : v, 200)}`).join("\n");
    const common = `${zl}
PLANNING DESK. Make a printable page of SAMPLE IDEAS (not a personal plan; the visitor is not being coached). Use general ranges only. No calories, grams, weights or weight-loss goals.
The visitor's answers:
${ans || "- (none)"}
`;
    if (kind === "routine") return common + `Make a sample weekly routine.
Shape:
{"title": "Sample Routine Ideas: <short goal name>",
 "intro": "2 sentences, starting 'Here are sample ideas...'",
 "week": [{"day": "Monday", "focus": "e.g. Upper body / Walk + stretch / Rest", "items": [{"name": "exercise name", "amount": "general range, e.g. 2-3 sets of 8-12", "cue": "one short form cue"}]}],
 "warmup": [{"name": "warm-up move", "amount": "e.g. 1 minute"}],
 "cooldown": [{"name": "stretch", "amount": "e.g. hold 20-30 seconds"}],
 "progression": ["Week 1 idea", "Week 2 idea", "Week 3 idea", "Week 4 idea"],
 "habits": ["3-5 healthy-habit ideas: water, sleep, steps, a simple food idea"],
 "note": "one sentence reminder to start easy and check with a doctor if they have any health concerns"}
If they listed equipment they have, use ONLY that equipment plus bodyweight moves; if they work out at home with no equipment, use bodyweight moves only. All 7 days in "week"; match the number of workout days to their answer and mark the others as Rest or Easy walk. 3-6 items on workout days, big muscles before small. Use only common exercises that match where they work out.`;
    if (kind === "challenge") return common + `Make a sample 30-day challenge.
Shape:
{"title": "Sample 30-Day <Challenge> Challenge",
 "intro": "2 sentences",
 "days": [{"day": 1, "target": "short daily target, e.g. 20-second plank"}],
 "tips": ["3-5 short tips"],
 "note": "one sentence safety reminder"}
Exactly 30 days, building gradually, with an easier day about every 7th day.`;
    if (kind === "desk") return common + `Make a sample desk stretch sheet.
Shape:
{"title": "Desk Stretch Sheet",
 "intro": "2 sentences",
 "stretches": [{"name": "stretch name", "how": "1-2 sentences", "hold": "e.g. 20 seconds each side"}],
 "schedule": "one or two sentences: an idea for when to do them during the day",
 "note": "one sentence: stop if anything hurts"}
8-10 stretches, focused on the trouble spots they picked, all doable at or beside a desk.`;
    if (kind === "swap") return common + `Make a sample food swap ideas sheet.
Shape:
{"title": "Food Swap Ideas",
 "intro": "2 sentences",
 "swaps": [{"instead": "a food or drink they like", "try": "a swap idea", "why": "one neutral sentence on what's different (more fiber, less added sugar, more protein, and so on)"}],
 "tips": ["3-4 short shopping or habit tips"],
 "note": "one sentence: swaps are ideas, and anyone with a health condition or allergy should check with a doctor or registered dietitian"}
8-12 swaps based on what they said they like. Never say any food is bad. Respect any eating style or allergy they mention.`;
    return null;
  }
  return null;
}

// ---------- tidy the AI output ----------
const cleanNamed = (a, n) => (Array.isArray(a) ? a : []).slice(0, n).map(x => ({ name: clip(x && x.name, 100), how: clip(x && x.how, 400) })).filter(x => x.name);
const cleanSecs = (a, n) => (Array.isArray(a) ? a : []).slice(0, n).map(d => ({ heading: clip(d && d.heading, 100), text: clip(d && d.text, 1400) })).filter(d => d.text);
const cleanLines = (a, n, ids) => (Array.isArray(a) ? a : []).slice(0, n).map(x => ({ id: GUIDES[String(x && x.id || "").toLowerCase()] ? String(x.id).toLowerCase() : ids[0], text: clip(x && x.text, 900) })).filter(x => x.text);
const NO_SOURCES = "Mr. Paul couldn't match this answer to strong sources this time, so treat it as general information and check with a trusted health source.";

function cleanCharts(list, retrieved) {
  const out = [];
  for (const c of Array.isArray(list) ? list : []) {
    if (!c || !c.sourceUrl) continue;
    const v = verifySources([{ title: c.sourceTitle, url: c.sourceUrl }], retrieved);
    if (!v.length) continue; // chart numbers must come from a page actually retrieved
    const bars = (Array.isArray(c.bars) ? c.bars : []).slice(0, 6).map(x => ({ label: clip(x && x.label, 40), value: Number(x && x.value) })).filter(x => x.label && isFinite(x.value));
    if (bars.length < 2) continue;
    out.push({ title: clip(c.title, 90), unit: clip(c.unit, 40), bars, note: clip(c.note, 300), source: v[0] });
    if (out.length >= 2) break;
  }
  return out;
}

// ---------- the handler ----------
module.exports = async (req, res) => {
  if (!(await require("./_lwgate")(req, res, "the-health-club"))) return;
  res.setHeader("Cache-Control", "no-store");
  if (req.method !== "POST") return res.status(405).json({ error: "POST only" });
  const origin = req.headers.origin || "";
  if (ALLOWED.length && origin && !ALLOWED.some(a => origin === a)) return res.status(403).json({ error: "This tool only works on its home website." });
  if (!process.env.ANTHROPIC_API_KEY) return res.status(500).json({ error: "Server is missing ANTHROPIC_API_KEY." });

  const ip = String(req.headers["x-forwarded-for"] || req.socket?.remoteAddress || "x").split(",")[0].trim();
  const b = req.body || {};
  const mode = String(b.mode || "");
  if (mode !== "diagram" && overLimit(ip, DAILY_LIMIT)) {
    return res.status(429).json({ error: "limit", message: "You've reached today's limit. Come back tomorrow and the coaches will be ready." });
  }
  const prompt = modePrompt(mode, b);
  if (!prompt) return res.status(400).json({ error: "Unknown mode or missing input" });

  try {
    // ---- quick modes (no web search)
    if (mode === "diagram") {
      let text = textOf(await anthropic([{ role: "user", content: prompt }], 0, DIAGRAM_MODEL, 7000));
      let svg = cleanSvg(text);
      if (!svg && DIAGRAM_MODEL !== MODEL) { text = textOf(await anthropic([{ role: "user", content: prompt }], 0, MODEL, 9000)); svg = cleanSvg(text); }
      const cap = (text.match(/CAPTION:\s*([^\n<]+)/i) || [])[1] || "";
      if (!svg) return res.status(200).json({ ok: false, message: "The diagram didn't come out right. Tap Try Again." });
      return res.status(200).json({ ok: true, svg, caption: clip(cap, 240) });
    }
    if (mode === "home") {
      const j = extractJson(textOf(await anthropic([{ role: "user", content: prompt }], 0, MODEL, 3500))) || {};
      const moves = (Array.isArray(j.moves) ? j.moves : []).slice(0, 14).map(x => ({ name: clip(x && x.name, 80), equipment: clip(x && x.equipment, 60), area: clip(x && x.area, 30), how: clip(x && x.how, 300) })).filter(x => x.name);
      if (!moves.length) return res.status(200).json({ ok: false, message: "Coach Mike couldn't put that list together just now. Please try again." });
      const c = j.circuit || {};
      return res.status(200).json({
        ok: true, intro: clip(j.intro, 500), setup: clipList(j.setup, 4, 300), moves,
        circuit: { name: clip(c.name, 80), rounds: clip(c.rounds, 160), items: (Array.isArray(c.items) ? c.items : []).slice(0, 6).map(x => ({ name: clip(x && x.name, 80), amount: clip(x && x.amount, 60) })).filter(x => x.name) },
        coachTip: clip(j.coachTip, 300), safety: clip(j.safety, 400)
      });
    }
    if (mode === "muscle") {
      const j = extractJson(textOf(await anthropic([{ role: "user", content: prompt }], 0, MODEL, 2500))) || {};
      return res.status(200).json({
        ok: true, muscle: clip(j.muscle, 80) || clip(b.muscle, 80), job: clip(j.job, 600), everyday: clip(j.everyday, 300),
        exercises: (Array.isArray(j.exercises) ? j.exercises : []).slice(0, 6).map(x => ({ name: clip(x && x.name, 80), equipment: clip(x && x.equipment, 40), how: clip(x && x.how, 300) })).filter(x => x.name),
        order: clip(j.order, 400), stretch: j.stretch && j.stretch.name ? { name: clip(j.stretch.name, 80), how: clip(j.stretch.how, 300) } : null, tip: clip(j.tip, 300)
      });
    }
    if (mode === "label") {
      const img = String(b.image || "");
      const m = img.match(/^data:(image\/(?:jpeg|png|webp|gif));base64,([A-Za-z0-9+/=]+)$/);
      if (!m || m[2].length > 4_000_000) return res.status(400).json({ error: "Please choose a clear photo (JPG or PNG) of the label." });
      const content = [{ type: "image", source: { type: "base64", media_type: m[1], data: m[2] } }, { type: "text", text: prompt }];
      const j = extractJson(textOf(await anthropic([{ role: "user", content }], 0, MODEL, 3000))) || {};
      if (j.ok === false || !Array.isArray(j.items)) return res.status(200).json({ ok: false, message: clip(j.message, 300) || "Coach Lisa couldn't read that one. Try a clear, close photo of the Nutrition Facts panel in good light." });
      return res.status(200).json({
        ok: true, product: clip(j.product, 120), servingSize: clip(j.servingSize, 300), servings: clip(j.servings, 300),
        items: j.items.slice(0, 16).map(x => ({ label: clip(x && x.label, 60), amount: clip(x && x.amount, 60), meaning: clip(x && x.meaning, 300) })).filter(x => x.label),
        sugarTsp: clip(j.sugarTsp, 200), dvNote: clip(j.dvNote, 300), ingredients: clipList(j.ingredients, 6, 300),
        terms: (Array.isArray(j.terms) ? j.terms : []).slice(0, 6).map(t => ({ term: clip(t && t.term, 60), meaning: clip(t && t.meaning, 300) })).filter(t => t.term),
        bigPicture: clip(j.bigPicture, 700), kidFact: clip(j.kidFact, 300), footer: FOOTER_NOTE
      });
    }
    if (mode === "plan") {
      const j = extractJson(textOf(await anthropic([{ role: "user", content: prompt }], 0, MODEL, 6000))) || {};
      const kind = b.kind;
      const out = { ok: true, kind, title: clip(j.title, 120), intro: clip(j.intro, 500), note: clip(j.note, 300), footer: FOOTER_NOTE };
      if (kind === "routine") {
        out.week = (Array.isArray(j.week) ? j.week : []).slice(0, 7).map(d => ({ day: clip(d && d.day, 20), focus: clip(d && d.focus, 60), items: (Array.isArray(d && d.items) ? d.items : []).slice(0, 7).map(x => ({ name: clip(x && x.name, 80), amount: clip(x && x.amount, 80), cue: clip(x && x.cue, 160) })).filter(x => x.name) }));
        const amt = a => (Array.isArray(a) ? a : []).slice(0, 6).map(x => ({ name: clip(x && x.name, 80), amount: clip(x && x.amount, 60) })).filter(x => x.name);
        out.warmup = amt(j.warmup); out.cooldown = amt(j.cooldown); out.progression = clipList(j.progression, 4, 300); out.habits = clipList(j.habits, 5, 200);
        if (!out.week.length) return res.status(200).json({ ok: false, message: "The coaches couldn't put that together just now. Please try again." });
      } else if (kind === "challenge") {
        out.days = (Array.isArray(j.days) ? j.days : []).slice(0, 30).map((d, i) => ({ day: i + 1, target: clip(d && d.target, 80) }));
        out.tips = clipList(j.tips, 5, 200);
        if (out.days.length < 10) return res.status(200).json({ ok: false, message: "The coaches couldn't put that together just now. Please try again." });
      } else if (kind === "desk") {
        out.stretches = (Array.isArray(j.stretches) ? j.stretches : []).slice(0, 10).map(x => ({ name: clip(x && x.name, 80), how: clip(x && x.how, 300), hold: clip(x && x.hold, 60) })).filter(x => x.name);
        out.schedule = clip(j.schedule, 300);
      } else if (kind === "swap") {
        out.swaps = (Array.isArray(j.swaps) ? j.swaps : []).slice(0, 12).map(x => ({ instead: clip(x && x.instead, 80), try: clip(x && x.try, 120), why: clip(x && x.why, 240) })).filter(x => x.instead && x.try);
        out.tips = clipList(j.tips, 4, 200);
      }
      return res.status(200).json(out);
    }

    // ---- researched modes
    const searches = mode === "debate" ? 5 : mode === "myth" ? 4 : mode === "ask" ? 2 : 3;
    const { text, retrieved } = await callClaude(prompt, searches);
    const j = extractJson(text) || {};
    const sources = verifySources(j.sources, retrieved);

    if (mode === "exercise" || mode === "machine") {
      if (j.ok === false) return res.status(200).json({ ok: false, message: clip(j.message, 400) || "The Health Club covers exercise, food and healthy habits. For other questions, try The Answer Man on NuSparkLearnWow.com." });
      const steps = cleanSteps(j.steps, 10);
      if (!steps.length) return res.status(200).json({ ok: false, message: "Coach Mike couldn't put that one together just now. Please try again in a minute." });
      return res.status(200).json({
        ok: true, title: clip(j.title, 120) || clip(b.name, 120), what: clip(j.what, 800), muscles: clipList(j.muscles, 6, 100), equipment: clip(j.equipment, 120),
        setup: clipList(j.setup, 6, 300), steps, breathing: clip(j.breathing, 300), mistakes: clipList(j.mistakes, 6, 300),
        easier: cleanNamed(j.easier, 3), harder: cleanNamed(j.harder, 3), sample: clip(j.sample, 300), safety: clip(j.safety, 400),
        coachTip: clip(j.coachTip, 300), deeper: cleanSecs(j.deeper, 3), followups: clipList(j.followups, 3, 140),
        sources, sourceNote: sources.length ? "" : NO_SOURCES, footer: FOOTER_NOTE
      });
    }
    if (mode === "reallife") {
      if (j.ok === false) return res.status(200).json({ ok: false, message: clip(j.message, 400) || "Try an everyday goal like carrying groceries or climbing stairs." });
      const allowed = (Array.isArray(b.muscles) ? b.muscles : []).map(x => String(x));
      const muscles = (Array.isArray(j.muscles) ? j.muscles : []).slice(0, 6).map(x => ({ muscle: clip(x && x.muscle, 40), why: clip(x && x.why, 300) })).filter(x => x.muscle && (!allowed.length || allowed.includes(x.muscle)));
      const moves = (Array.isArray(j.moves) ? j.moves : []).slice(0, 6).map(x => ({ name: clip(x && x.name, 80), level: oneOf(String(x && x.level || ""), ["Easier", "Medium", "Harder"], "Medium"), where: oneOf(String(x && x.where || ""), ["Home", "Gym", "Anywhere"], "Anywhere"), how: clip(x && x.how, 300) })).filter(x => x.name);
      if (!moves.length) return res.status(200).json({ ok: false, message: "Coach Mike couldn't break that one down just now. Please try again." });
      return res.status(200).json({
        ok: true, title: clip(j.title, 140) || clip(b.goal, 140), why: clip(j.why, 700), muscles, moves, practice: clip(j.practice, 400),
        wow: sources.length ? clip(j.wow, 400) : "", coachTip: clip(j.coachTip, 300), safety: clip(j.safety, 400),
        sources, sourceNote: sources.length ? "" : NO_SOURCES, footer: FOOTER_NOTE
      });
    }
    if (mode === "topic") {
      if (j.ok === false) return res.status(200).json({ ok: false, message: clip(j.message, 400) || "The Health Club covers exercise, food and healthy habits. For other questions, try The Answer Man on NuSparkLearnWow.com." });
      const sections = cleanSecs(j.sections, 5);
      if (!sections.length && !j.intro) return res.status(200).json({ ok: false, message: "Coach Lisa couldn't put that one together just now. Please try again in a minute." });
      return res.status(200).json({
        ok: true, title: clip(j.title, 140) || clip(b.topic, 140), intro: clip(j.intro, 900), sections, tryThis: clipList(j.tryThis, 5, 300),
        myths: (Array.isArray(j.myths) ? j.myths : []).slice(0, 3).map(x => ({ myth: clip(x && x.myth, 200), truth: clip(x && x.truth, 500) })).filter(x => x.myth),
        research: clip(j.research, 600), doctorNote: clip(j.doctorNote, 300), followups: clipList(j.followups, 3, 140),
        sources, sourceNote: sources.length ? "" : NO_SOURCES, footer: FOOTER_NOTE
      });
    }
    if (mode === "myth") {
      const cards = (Array.isArray(j.cards) ? j.cards : []).slice(0, 6).map(c => {
        const src = c && c.source ? verifySources([c.source], retrieved) : [];
        return { statement: clip(c && c.statement, 200), answer: oneOf(String(c && c.answer || "").toLowerCase(), ["myth", "fact", "partly"], "partly"), explain: clip(c && c.explain, 600), source: src[0] || null };
      }).filter(c => c.statement && c.explain);
      if (!cards.length) return res.status(200).json({ ok: false, message: "The wall is being repainted. Tap New Cards to try again." });
      return res.status(200).json({ ok: true, cards });
    }
    if (mode === "debate") {
      if (j.ok === false) return res.status(200).json({ ok: false, message: clip(j.message, 400) || "The Great Debate covers exercise, food and healthy habits. Try another question." });
      const ids = ["mike", "lisa", "data"];
      const opening = cleanLines(j.opening, 3, ids);
      if (!opening.length) return res.status(200).json({ ok: false, message: "The panel couldn't get started just now. Please try again in a minute." });
      const pos = j.positions || {};
      const w = j.wrap || {};
      return res.status(200).json({
        ok: true, question: clip(j.question, 220) || clip(b.question, 220),
        positions: { mike: clip(pos.mike, 90), lisa: clip(pos.lisa, 90), data: clip(pos.data, 90) },
        opening, rounds: (Array.isArray(j.rounds) ? j.rounds : []).slice(0, 2).map(r => cleanLines(r, 3, ids)).filter(r => r.length),
        charts: cleanCharts(j.charts, retrieved),
        wrap: { best: (Array.isArray(w.best) ? w.best : []).slice(0, 3).map(x => ({ id: GUIDES[x && x.id] ? x.id : "data", point: clip(x && x.point, 300) })).filter(x => x.point), research: clip(w.research, 900), bottomLine: clip(w.bottomLine, 300) },
        vote: clipList(j.vote, 3, 60), sources, sourceNote: sources.length ? "" : NO_SOURCES, footer: FOOTER_NOTE
      });
    }
    if (mode === "ask") {
      const def = GUIDES[b.to] ? b.to : "lisa";
      const replies = cleanLines(j.replies, 2, [def]).map(r => ({ id: r.id, text: clip(r.text, 1600) }));
      if (!replies.length) return res.status(200).json({ replies: [{ id: def, text: "The coaches couldn't confirm an answer to that one. Try asking it a different way." }], clarify: "", sources: [] });
      return res.status(200).json({ replies, clarify: clip(j.clarify, 200), sources });
    }
    return res.status(400).json({ error: "Unknown mode" });
  } catch (e) {
    console.error(e);
    return res.status(500).json({ error: "The coaches couldn't reach the club just now. Please try again in a minute." });
  }
};
