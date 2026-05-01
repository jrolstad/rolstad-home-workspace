# Ambient Weather MCP Server

Create an MCP server that can run locally or be hosted as a Lambda function, Cloudflare Worker, or other lightweight hosting tool.
This MCP server needs to wrap the Ambient Weather API.  The api is documented at https://ambientweather.docs.apiary.io/#

A golang SDK for this API is in the ambientweather-api-helper repository.
A popular python SDK that wraps the API is at https://github.com/bachya/aioambient
An existing MCP server is at https://github.com/NanaGyamfiPrempeh30/ambient-weather-mcp