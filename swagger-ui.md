---
title: "Interactive API Explorer"
description: "Use the Swagger UI below to explore and test API endpoints directly in your browser."
---

Use the Swagger UI below to explore and test API endpoints directly in your browser.

<iframe 
  src="/swagger-ui.html" 
  style="width: 100%; height: 800px; border: none; border-radius: 4px;"
  title="Swagger UI"
  sandbox="allow-scripts allow-same-origin allow-forms">
</iframe>

## Using the API Explorer

### Authentication

1. Click the **Authorize** button at the top right
2. Enter your bearer token in the format: `Bearer YOUR_TOKEN_HERE`
3. Click **Authorize** to apply to all requests

### Making Requests

1. Expand an endpoint section (e.g., "workspaces")
2. Click on a specific operation (e.g., "GET /api/v1/workspaces")
3. Click **Try it out**
4. Fill in any required parameters
5. Click **Execute**
6. View the response below

### Response Codes

The explorer shows:
- **Curl command** - Copy this to use from the command line
- **Request URL** - The full URL that was called
- **Response body** - The JSON response
- **Response headers** - Including ETag for updates

## Alternative: Direct Link

If the embedded view doesn't work, open the API explorer directly:

[Open Swagger UI in New Tab](/swagger-ui-standalone.html)

## Notes

- The API explorer connects to the production API by default
- All requests use your authenticated session
- Be careful with DELETE operations - they affect real resources
- Rate limits apply to requests made through the explorer

