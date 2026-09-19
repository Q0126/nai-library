const ALLOWED_PATHS = new Set([
  "/artists.json",
  "/tags.json",
  "/posts.json"
]);

const CORS_HEADERS = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type, Accept"
};

function jsonResponse(data, status = 200) {
  return new Response(JSON.stringify(data), {
    status,
    headers: {
      ...CORS_HEADERS,
      "Content-Type": "application/json; charset=utf-8"
    }
  });
}

export default {
  async fetch(request) {
    if (request.method === "OPTIONS") {
      return new Response(null, {
        status: 204,
        headers: CORS_HEADERS
      });
    }

    if (request.method !== "GET") {
      return jsonResponse({ error: "Method not allowed" }, 405);
    }

    const requestUrl = new URL(request.url);
    const rawUrl = requestUrl.searchParams.get("url");

    if (!rawUrl) {
      return jsonResponse({ error: "Missing Danbooru URL" }, 400);
    }

    let target;
    try {
      target = new URL(rawUrl);
    } catch {
      return jsonResponse({ error: "Invalid Danbooru URL" }, 400);
    }

    if (
      target.protocol !== "https:" ||
      target.hostname !== "danbooru.donmai.us" ||
      !ALLOWED_PATHS.has(target.pathname)
    ) {
      return jsonResponse({ error: "Target not allowed" }, 403);
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
        return jsonResponse({
          error: "Danbooru request failed",
          upstreamStatus: upstream.status
        }, 502);
      }

      if (!contentType.includes("application/json")) {
        return jsonResponse({
          error: "Danbooru returned a verification page instead of JSON"
        }, 502);
      }

      return new Response(body, {
        status: 200,
        headers: {
          ...CORS_HEADERS,
          "Content-Type": "application/json; charset=utf-8",
          "Cache-Control": "s-maxage=300, stale-while-revalidate=600"
        }
      });
    } catch (error) {
      return jsonResponse({
        error: "Danbooru connection failed",
        message: String(error && error.message ? error.message : error)
      }, 502);
    }
  }
};
