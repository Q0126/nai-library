const ALLOWED_PATHS = new Set([
  "/artists.json",
  "/tags.json",
  "/posts.json"
]);

module.exports = async function handler(req, res) {
  res.setHeader("Access-Control-Allow-Origin", "*");
  res.setHeader("Access-Control-Allow-Methods", "GET, OPTIONS");
  res.setHeader("Access-Control-Allow-Headers", "Content-Type, Accept");

  if (req.method === "OPTIONS") {
    return res.status(204).end();
  }

  if (req.method !== "GET") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const rawUrl = Array.isArray(req.query.url) ? req.query.url[0] : req.query.url;
  if (!rawUrl) {
    return res.status(400).json({ error: "Missing Danbooru URL" });
  }

  let target;
  try {
    target = new URL(rawUrl);
  } catch {
    return res.status(400).json({ error: "Invalid Danbooru URL" });
  }

  if (
    target.protocol !== "https:" ||
    target.hostname !== "danbooru.donmai.us" ||
    !ALLOWED_PATHS.has(target.pathname)
  ) {
    return res.status(403).json({ error: "Target not allowed" });
  }

  try {
    const upstream = await fetch(target.toString(), {
      headers: {
        "Accept": "application/json",
        "User-Agent": "NAI-Artist-Database/1.0 (Danbooru search frontend)"
      },
      redirect: "follow"
    });

    const contentType = upstream.headers.get("content-type") || "";
    const body = await upstream.text();

    if (!upstream.ok) {
      return res.status(502).json({
        error: "Danbooru request failed",
        upstreamStatus: upstream.status
      });
    }

    if (!contentType.includes("application/json")) {
      return res.status(502).json({
        error: "Danbooru returned a verification page instead of JSON"
      });
    }

    res.setHeader("Content-Type", "application/json; charset=utf-8");
    res.setHeader("Cache-Control", "s-maxage=300, stale-while-revalidate=600");
    return res.status(200).send(body);
  } catch (error) {
    return res.status(502).json({
      error: "Danbooru connection failed",
      message: String(error && error.message ? error.message : error)
    });
  }
};
