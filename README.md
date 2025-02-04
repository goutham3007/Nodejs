# Nodejs


---

New API endpoint:

GET /api/analytics/topic/{topic}: This endpoint will return analytics specific to a topic.
Database changes:

You’ll need to link a topic to the shortened URL and analytics.
You can store each shortened URL under a specific topic and associate analytics with that topic.
Analytics Logic:

We'll store analytics by topic and track click count, user agent, IP, and geo-location for each topic.
Updated Implementation:
1. Database Schema (Updated):
Topic Table: To store information related to the topics.
Columns: id, topic_name, created_at
URL Table: Store information about shortened URLs.
Columns: id, original_url, shortened_url, topic_id, created_at
Analytics Table: Store analytics for each URL.
Columns: id, url_id, click_count, user_agent, ip_address, geo_location, timestamp
Backend Implementation (Node.js/Express):
a. API to Shorten URL with Topic:
js
Copy
app.post('/shorten', async (req, res) => {
  const { url, customAlias, topic } = req.body;

  // Validate that the topic exists (for example, check in the database)
  const topicRecord = await db.query('SELECT * FROM topics WHERE topic_name = $1', [topic]);

  if (!topicRecord) {
    return res.status(400).json({ error: 'Topic not found' });
  }

  let alias = customAlias || generateShortUrl(url);

  if (urlDatabase[alias]) {
    return res.status(400).json({ error: 'Alias already taken' });
  }

  // Store URL with associated topic
  const topicId = topicRecord.id; // Assuming we got the topic ID from the topic query
  urlDatabase[alias] = { url, topicId, clicks: 0 };

  res.json({ shortenedUrl: `http://localhost:3000/${alias}` });
});
b. Analytics for Topic Endpoint (/api/analytics/topic/{topic}):
js
Copy
app.get('/api/analytics/topic/:topic', async (req, res) => {
  const topic = req.params.topic;

  // Retrieve topic information from the database
  const topicRecord = await db.query('SELECT * FROM topics WHERE topic_name = $1', [topic]);

  if (!topicRecord) {
    return res.status(404).json({ error: 'Topic not found' });
  }

  // Retrieve all URLs associated with the topic
  const urls = await db.query('SELECT * FROM urls WHERE topic_id = $1', [topicRecord.id]);

  if (urls.length === 0) {
    return res.status(404).json({ error: 'No URLs found for this topic' });
  }

  // Gather analytics for each URL
  const analytics = [];
  
  for (let url of urls) {
    const stats = await db.query('SELECT * FROM analytics WHERE url_id = $1', [url.id]);

    let totalClicks = 0;
    let userAgentStats = {};
    let geoLocationStats = {};

    // Aggregate analytics
    stats.forEach(stat => {
      totalClicks += stat.click_count;
      userAgentStats[stat.user_agent] = (userAgentStats[stat.user_agent] || 0) + 1;
      geoLocationStats[stat.geo_location] = (geoLocationStats[stat.geo_location] || 0) + 1;
    });

    analytics.push({
      url: url.shortened_url,
      totalClicks,
      userAgentStats,
      geoLocationStats
    });
  }

  res.json({ topic: topic, analytics });
});
c. Updating the Click Analytics on Redirect (/alias endpoint):
js
Copy
app.get('/:alias', async (req, res) => {
  const alias = req.params.alias;

  if (!urlDatabase[alias]) {
    return res.status(404).send('URL not found');
  }

  const originalUrl = urlDatabase[alias].url;
  const topicId = urlDatabase[alias].topicId;
  
  // Increment click count in database (you can also track the IP, user-agent, etc.)
  await db.query('INSERT INTO analytics (url_id, click_count, user_agent, ip_address, geo_location) VALUES ($1, 1, $2, $3, $4)', [
    urlDatabase[alias].id,
    req.headers['user-agent'],
    req.ip,
    'unknown' // You can use a geolocation library to find the real location from IP
  ]);

  // Redirect to original URL
  res.redirect(originalUrl);
});
Example Usage:
Create a shortened URL for a specific topic:

Request:

json
Copy
{
  "url": "https://example.com",
  "customAlias": "newtopic-url",
  "topic": "technology"
}
Response:

json
Copy
{
  "shortenedUrl": "http://localhost:3000/newtopic-url"
}
Get Analytics for the Topic "technology":

Request:

http
Copy
GET /api/analytics/topic/technology
Response:

json
Copy
{
  "topic": "technology",
  "analytics": [
    {
      "url": "http://localhost:3000/newtopic-url",
      "totalClicks": 100,
      "userAgentStats": {
        "Mozilla/5.0": 90,
        "Chrome/91.0": 10
      },
      "geoLocationStats": {
        "USA": 80,
        "India": 20
      }
    }
  ]
}
3. Rate Limiting:
We can add rate limiting on this /api/analytics/topic/{topic} endpoint to prevent abuse. For example, limit how often a user can fetch analytics for a topic.

js
Copy
const limiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 3, // Limit each user to 3 requests per minute
  message: "Too many requests, please try again later."
});

app.use('/api/analytics/topic/', limiter); // Apply rate limiting to the analytics endpoint
Final Considerations:
User authentication: Implement login/signup for users, so each user has their own set of shortened URLs and analytics.
Topic management: Create an admin panel to manage topics or let users define their own topics when shortening URLs.
Geo-Location: You can use an external service to get the user's location based on their IP.
Security: Ensure the alias and URL data are sanitized properly to avoid security vulnerabilities.
This updated structure aligns with your requirement for the endpoint /api/analytics/topic/{topic}, where you can fetch analytics specific to a given topic.







