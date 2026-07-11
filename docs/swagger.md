# API Reference (Swagger)

Interactive documentation for the Book Platform API. Expand any endpoint to see its
parameters, request body, and example responses. (To run live requests, the WordPress
backend must be running at `http://localhost:8080`.)

<link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css" />
<div id="swagger-ui"></div>
<script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
<script>
  window.addEventListener("DOMContentLoaded", function () {
    window.ui = SwaggerUIBundle({
      url: "openapi.yaml",
      dom_id: "#swagger-ui",
      deepLinking: true,
      docExpansion: "list",
    });
  });
</script>
