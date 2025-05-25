openapi: 3.1.0
info:
  title: GitHub Obsidian API
  version: 1.0.1
  description: >
    API to manage notes in an Obsidian vault stored in a GitHub repository, including error logging functionality.
    **Note:** 
    - The `sha` field is optional. Omit it when creating a new file; if updating an existing file, retrieve the latest SHA via a GET call before sending.
servers:
  - url: https://api.github.com

paths:
  /repos/giovannilucastefano/Obsidian/contents:
    get:
      operationId: listNotesRoot
      summary: List all notes in the root folder
      description: Retrieves all files and folders in the repository root.
      x-openai-isConsequential: false
      security:
        - github_auth: []
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/FileResponse'

  /repos/giovannilucastefano/Obsidian/contents/{folderPath}:
    get:
      operationId: listNotesInFolder
      summary: List notes in a specific folder
      description: Retrieves a list of notes from a specific folder in the repository.
      x-openai-isConsequential: false
      parameters:
        - name: folderPath
          in: path
          required: true
          schema:
            type: string
          description: Path to the folder (e.g., "00 - Inbox").
      security:
        - github_auth: []
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/FileResponse'

  /repos/giovannilucastefano/Obsidian/contents/{path}:
    get:
      operationId: getNote
      summary: Read a note
      description: Retrieves the content of a Markdown note from the repository.
      x-openai-isConsequential: false
      parameters:
        - name: path
          in: path
          required: true
          schema:
            type: string
            pattern: ".*\\.md$"
          description: Path to the Markdown note (e.g., "00 - Inbox/project.md").
      security:
        - github_auth: []
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/FileResponse'

    put:
      operationId: createOrUpdateNote
      summary: Create or update a note
      description: >
        Creates or updates a note in the repository. Use the `contentType` parameter for plain text or table updates.
        **Important:**
        - Provide Base64-encoded content as a continuous string.
        - Omit `sha` when creating a new file. When updating, retrieve the latest SHA via a GET call before sending.
      x-openai-isConsequential: false
      parameters:
        - name: path
          in: path
          required: true
          schema:
            type: string
            pattern: ".*\\.md$"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - message
                - content
              properties:
                message:
                  type: string
                  description: >
                    A commit message describing the update (e.g., "Added an empty row to the table").
                  example: "Added an empty row to the table"
                contentType:
                  type: string
                  enum: [text, table]
                  description: Specify whether the content is plain text or a Markdown table.
                content:
                  type: string
                  description: >
                    Base64-encoded string must:
                    - Consist solely of valid Base64 characters (A-Z, a-z, 0-9, +, /, =)
                    - Be a continuous string (no newlines)
                    - Have a length that is a multiple of 4.
                  pattern: "^(?:[A-Za-z0-9+/=]{4})*$"
                sha:
                  type: string

      responses:
        '200':
          description: Successfully updated the note
        '422':
          description: >
            Error due to invalid content, SHA mismatch, or Base64 encoding issue.
            - SHA mismatch: Retrieve the latest file version and retry.
            - Base64 issue: Check for invalid characters or encoding errors.

  /repos/giovannilucastefano/Obsidian/contents/03%20-%20Resources/Service%20Notes/api-errors.md:
    post:
      operationId: logApiError
      summary: Log an API error
      description: >
        Appends error details to the `api-errors.md` file in the `03 - Resources/Service Notes` folder.
        Creates the file if it doesn't exist.
      x-openai-isConsequential: false
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - endpoint
                - operation
                - errorMessage
                - timestamp
              properties:
                endpoint:
                  type: string
                  description: The API endpoint where the error occurred.
                operation:
                  type: string
                  description: The operation being performed (e.g., createOrUpdateNote).
                errorMessage:
                  type: string
                  description: The error message returned by the API.
                payload:
                  type: object
                  description: The payload sent during the API call (if available).
                path:
                  type: string
                  description: Path to the file associated with the error (if applicable).
                timestamp:
                  type: string
                  description: ISO 8601 timestamp of when the error occurred.
                  example: "2025-02-08T14:32:00Z"
      responses:
        '200':
          description: Error successfully logged
        '201':
          description: Log file created and error logged
        '400':
          description: Invalid error details provided
      examples:
        shaMismatch:
          summary: SHA mismatch error example
          value:
            endpoint: "/repos/giovannilucastefano/Obsidian/contents/{path}"
            operation: "createOrUpdateNote"
            errorMessage: "SHA mismatch. The file has been updated elsewhere."
            path: "00 - Inbox/Weekly Habits Test.md"
            timestamp: "2025-02-08T14:32:00Z"
        base64Error:
          summary: Base64 encoding issue example
          value:
            endpoint: "/repos/giovannilucastefano/Obsidian/contents/{path}"
            operation: "createOrUpdateNote"
            errorMessage: "Invalid Base64 encoding. String length is not a multiple of 4."
            path: "01 - Projects/ProjectNote.md"
            timestamp: "2025-02-08T14:40:00Z"

  /search/code:
    get:
      operationId: searchNotes
      summary: Search notes
      description: Searches for notes in the repository using GitHub's code search API.
      x-openai-isConsequential: false
      parameters:
        - name: q
          in: query
          required: true
          schema:
            type: string
          example: "searchterm repo:giovannilucastefano/Obsidian extension:md"
          description: GitHub search query to find Markdown files.
      security:
        - github_auth: []
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SearchResponse'

components:
  securitySchemes:
    github_auth:
      type: apiKey
      in: header
      name: Authorization

  schemas:
    FileUpdate:
      type: object
      required:
        - message
        - content
      properties:
        message:
          type: string
          example: "Added an empty row to the table"
          description: Commit message for the update.
        content:
          type: string
          description: >
            Base64-encoded content of the file. The encoded string must be continuous (no newlines)
            and consist solely of valid Base64 characters (A-Z, a-z, 0-9, +, /, =) with a length that is a multiple of 4.
          pattern: "^(?:[A-Za-z0-9+/=]{4})*$"
        sha:
          type: string
          description: >
            (Optional) The SHA of the file being updated. Omit for new files.
        contentType:
          type: string
          enum: [text, table]
          description: Specify whether the content is plain text or a Markdown table.

    FileResponse:
      type: object
      properties:
        type:
          type: string
        encoding:
          type: string
          description: "Should be 'base64' if content is encoded."
        size:
          type: integer
        name:
          type: string
        path:
          type: string
        content:
          type: string
          description: "Base64-encoded file content. Decode to view original text."
        sha:
          type: string
        url:
          type: string
        git_url:
          type: string
        html_url:
          type: string
        download_url:
          type: string

    SearchResponse:
      type: object
      properties:
        total_count:
          type: integer
