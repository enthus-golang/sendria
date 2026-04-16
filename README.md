# sendria

> [!WARNING]
> **This project is no longer maintained.**
>
> We originally built this library to test SMTP-based email functionality in our Go services against a [Sendria](https://github.com/msztolcman/sendria) instance. We have since moved away from this approach and recommend one of the following alternatives instead:
>
> - **[github.com/uponusolutions/go-smtp](https://github.com/uponusolutions/go-smtp)** — a pure-Go ESMTP client and server library. Its `tester` package provides an embeddable in-process SMTP server with a mail map, which removes the need for an external service like Sendria entirely. This is what we use now for testing SMTP in Go.
> - **[smtp4dev](https://github.com/rnwood/smtp4dev)** — if you still want a standalone capture server (like Sendria), smtp4dev is a well-maintained alternative that supports significantly more standard APIs (IMAP, proper REST, web UI, etc.) than Sendria does.
>
> The repository is kept available for existing users, but no further updates, bug fixes, or releases are planned.

[![Go Reference](https://pkg.go.dev/badge/github.com/enthus-golang/sendria.svg)](https://pkg.go.dev/github.com/enthus-golang/sendria)
[![CI](https://github.com/enthus-golang/sendria/actions/workflows/go.yml/badge.svg)](https://github.com/enthus-golang/sendria/actions/workflows/go.yml)
[![Go Report Card](https://goreportcard.com/badge/github.com/enthus-golang/sendria)](https://goreportcard.com/report/github.com/enthus-golang/sendria)
[![codecov](https://codecov.io/gh/enthus-golang/sendria/branch/main/graph/badge.svg)](https://codecov.io/gh/enthus-golang/sendria)
[![License](https://img.shields.io/github/license/enthus-golang/sendria)](https://github.com/enthus-golang/sendria/blob/main/LICENSE)
[![Release](https://img.shields.io/github/release/enthus-golang/sendria.svg)](https://github.com/enthus-golang/sendria/releases/latest)

Go testing library for [Sendria](https://github.com/msztolcman/sendria) - making email testing simple and reliable.

## Overview

`sendria` is a Go client library specifically designed for testing email functionality in your applications. It integrates with Sendria, an SMTP server that captures emails instead of sending them, making it perfect for:

- ✅ **Unit and integration testing** of email features
- ✅ **Local development** without sending real emails
- ✅ **CI/CD pipelines** with email verification
- ✅ **Debugging** email content and formatting

## Why Sendria for Testing?

- **No real emails sent** - All emails are captured locally
- **Full email inspection** - View headers, body, attachments
- **REST API access** - Programmatically verify email content
- **Easy cleanup** - Clear messages between tests
- **Docker ready** - Simple integration with CI/CD

## Installation

```bash
go get github.com/enthus-golang/sendria
```

## Quick Start: Testing Email

Here's a complete example of testing email functionality:

```go
package myapp_test

import (
    "testing"
    "time"
    
    "github.com/enthus-golang/sendria"
)

func TestPasswordResetEmail(t *testing.T) {
    // Create Sendria client
    client := sendria.NewClient("http://localhost:1080")
    
    // Clear any existing messages
    if err := client.DeleteAllMessages(); err != nil {
        t.Fatal(err)
    }
    
    // Trigger password reset in your app
    err := YourApp.SendPasswordResetEmail("user@example.com")
    if err != nil {
        t.Fatal(err)
    }
    
    // Wait for email to arrive
    time.Sleep(100 * time.Millisecond)
    
    // Verify email was sent
    messages, err := client.ListMessages(1, 10)
    if err != nil {
        t.Fatal(err)
    }
    
    if len(messages.Messages) != 1 {
        t.Fatalf("Expected 1 email, got %d", len(messages.Messages))
    }
    
    // Verify email content
    msg := messages.Messages[0]
    if msg.Subject != "Password Reset Request" {
        t.Errorf("Wrong subject: %s", msg.Subject)
    }
    
    if msg.To[0].Email != "user@example.com" {
        t.Errorf("Wrong recipient: %s", msg.To[0].Email)
    }
    
    // Check email body contains reset link
    body, err := client.GetMessagePlain(msg.ID)
    if err != nil {
        t.Fatal(err)
    }
    
    if !strings.Contains(body, "https://example.com/reset?token=") {
        t.Error("Email missing reset link")
    }
}
```

## Testing Guide

### Setting Up Sendria for Tests

#### Using Docker (Recommended)

Create a `docker-compose.test.yml`:

```yaml
version: '3.8'
services:
  sendria:
    image: msztolcman/sendria:latest
    ports:
      - "1025:1025"  # SMTP port
      - "1080:1080"  # HTTP API port
    command: >
      sendria 
      --smtp-ip=0.0.0.0
      --http-ip=0.0.0.0
      --db=/tmp/sendria.db
      --smtp-auth=no
```

Run before tests:
```bash
docker-compose -f docker-compose.test.yml up -d
```

#### Local Installation

```bash
pip install sendria
sendria --db /tmp/sendria.db
```

### Test Helpers

Create reusable test helpers in `email_test_helper.go`:

```go
package testhelpers

import (
    "testing"
    "time"
    
    "github.com/enthus-golang/sendria"
)

// EmailTestClient wraps Sendria client with test helpers
type EmailTestClient struct {
    *sendria.Client
    t *testing.T
}

// NewEmailTestClient creates a test-friendly email client
func NewEmailTestClient(t *testing.T) *EmailTestClient {
    t.Helper()
    
    client := sendria.NewClient("http://localhost:1080")
    
    // Clear messages at start
    if err := client.DeleteAllMessages(); err != nil {
        t.Fatalf("Failed to clear messages: %v", err)
    }
    
    // Ensure cleanup after test
    t.Cleanup(func() {
        _ = client.DeleteAllMessages()
    })
    
    return &EmailTestClient{
        Client: client,
        t:      t,
    }
}

// WaitForEmails waits for expected number of emails
func (c *EmailTestClient) WaitForEmails(count int, timeout time.Duration) []sendria.Message {
    c.t.Helper()
    
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        messages, err := c.ListMessages(1, 10)
        if err != nil {
            c.t.Fatalf("Failed to list messages: %v", err)
        }
        
        if len(messages.Messages) >= count {
            return messages.Messages[:count]
        }
        
        time.Sleep(50 * time.Millisecond)
    }
    
    c.t.Fatalf("Timeout waiting for %d emails", count)
    return nil
}

// AssertEmailSent verifies an email was sent to recipient
func (c *EmailTestClient) AssertEmailSent(to, subject string) *sendria.Message {
    c.t.Helper()
    
    messages := c.WaitForEmails(1, 2*time.Second)
    msg := messages[0]
    
    if msg.To[0].Email != to {
        c.t.Errorf("Expected recipient %s, got %s", to, msg.To[0].Email)
    }
    
    if msg.Subject != subject {
        c.t.Errorf("Expected subject %q, got %q", subject, msg.Subject)
    }
    
    return &msg
}
```

### Table-Driven Tests

Test multiple email scenarios efficiently:

```go
func TestEmailNotifications(t *testing.T) {
    client := testhelpers.NewEmailTestClient(t)
    
    tests := []struct {
        name     string
        event    string
        user     User
        expected struct {
            subject  string
            template string
            contains []string
        }
    }{
        {
            name:  "welcome email",
            event: "user.created",
            user:  User{Email: "new@example.com", Name: "Alice"},
            expected: struct {
                subject  string
                template string
                contains []string
            }{
                subject:  "Welcome to Our App!",
                template: "welcome",
                contains: []string{"Hi Alice", "Get started"},
            },
        },
        {
            name:  "payment received",
            event: "payment.success",
            user:  User{Email: "customer@example.com", Name: "Bob"},
            expected: struct {
                subject  string
                template string
                contains []string
            }{
                subject:  "Payment Received - Thank You!",
                template: "payment_success",
                contains: []string{"$99.99", "Order #12345"},
            },
        },
        {
            name:  "subscription expiring",
            event: "subscription.expiring",
            user:  User{Email: "subscriber@example.com", Name: "Carol"},
            expected: struct {
                subject  string
                template string
                contains []string
            }{
                subject:  "Your Subscription is Expiring Soon",
                template: "subscription_reminder",
                contains: []string{"expires in 7 days", "Renew now"},
            },
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Clear messages for each test
            if err := client.DeleteAllMessages(); err != nil {
                t.Fatal(err)
            }
            
            // Trigger notification
            err := YourApp.SendNotification(tt.event, tt.user)
            if err != nil {
                t.Fatal(err)
            }
            
            // Verify email
            msg := client.AssertEmailSent(tt.user.Email, tt.expected.subject)
            
            // Get email content
            body, err := client.GetMessagePlain(msg.ID)
            if err != nil {
                t.Fatal(err)
            }
            
            // Verify content
            for _, text := range tt.expected.contains {
                if !strings.Contains(body, text) {
                    t.Errorf("Email missing expected text: %q", text)
                }
            }
        })
    }
}
```

### Testing HTML Emails

```go
func TestHTMLEmailTemplate(t *testing.T) {
    client := testhelpers.NewEmailTestClient(t)
    
    // Send HTML email
    err := YourApp.SendNewsletter("subscriber@example.com")
    if err != nil {
        t.Fatal(err)
    }
    
    msg := client.AssertEmailSent("subscriber@example.com", "Monthly Newsletter")
    
    // Verify HTML content
    html, err := client.GetMessageHTML(msg.ID)
    if err != nil {
        t.Fatal(err)
    }
    
    // Check HTML structure
    if !strings.Contains(html, `<div class="newsletter">`) {
        t.Error("Missing newsletter container")
    }
    
    if !strings.Contains(html, `<a href="https://example.com/unsubscribe"`) {
        t.Error("Missing unsubscribe link")
    }
    
    // Verify plain text alternative
    plain, err := client.GetMessagePlain(msg.ID)
    if err != nil {
        t.Fatal(err)
    }
    
    if plain == "" {
        t.Error("Missing plain text version")
    }
}
```

### Testing Attachments

```go
func TestEmailWithAttachment(t *testing.T) {
    client := testhelpers.NewEmailTestClient(t)
    
    // Send email with PDF invoice
    err := YourApp.SendInvoice("customer@example.com", "INV-001")
    if err != nil {
        t.Fatal(err)
    }
    
    msg := client.AssertEmailSent("customer@example.com", "Invoice INV-001")
    
    // Get full message with attachments
    fullMsg, err := client.GetMessage(msg.ID)
    if err != nil {
        t.Fatal(err)
    }
    
    // Verify attachment
    if len(fullMsg.Attachments) != 1 {
        t.Fatalf("Expected 1 attachment, got %d", len(fullMsg.Attachments))
    }
    
    att := fullMsg.Attachments[0]
    if att.Filename != "invoice_INV-001.pdf" {
        t.Errorf("Wrong filename: %s", att.Filename)
    }
    
    if att.ContentType != "application/pdf" {
        t.Errorf("Wrong content type: %s", att.ContentType)
    }
    
    // Download and verify attachment
    data, err := client.GetAttachment(msg.ID, att.CID)
    if err != nil {
        t.Fatal(err)
    }
    
    if len(data) == 0 {
        t.Error("Empty attachment")
    }
}
```

### Testing Bulk Emails

```go
func TestBulkEmailSending(t *testing.T) {
    client := testhelpers.NewEmailTestClient(t)
    
    recipients := []string{
        "user1@example.com",
        "user2@example.com",
        "user3@example.com",
    }
    
    // Send bulk emails
    err := YourApp.SendBulkAnnouncement(recipients, "Important Update")
    if err != nil {
        t.Fatal(err)
    }
    
    // Wait for all emails
    messages := client.WaitForEmails(len(recipients), 5*time.Second)
    
    // Verify each recipient got an email
    receivedEmails := make(map[string]bool)
    for _, msg := range messages {
        if msg.Subject == "Important Update" {
            receivedEmails[msg.To[0].Email] = true
        }
    }
    
    for _, recipient := range recipients {
        if !receivedEmails[recipient] {
            t.Errorf("No email sent to %s", recipient)
        }
    }
}
```

## CI/CD Integration

### GitHub Actions

`.github/workflows/test.yml`:

```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      sendria:
        image: msztolcman/sendria:v2.2.2.0
        ports:
          - 1025:1025
          - 1080:1080
        options: >-
          --health-cmd "curl -f http://localhost:1080/api/messages/ || exit 1"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      
      - name: Run tests
        env:
          SENDRIA_URL: http://localhost:1080
          SMTP_HOST: localhost:1025
        run: go test ./... -v
```

### GitLab CI

`.gitlab-ci.yml`:

```yaml
test:
  image: golang:1.21
  
  services:
    - name: msztolcman/sendria:latest
      alias: sendria
      command: ["sendria", "--smtp-ip=0.0.0.0", "--http-ip=0.0.0.0"]
  
  variables:
    SENDRIA_URL: "http://sendria:1080"
    SMTP_HOST: "sendria:1025"
  
  script:
    - go test ./... -v
```

## Best Practices

### 1. Test Isolation

Always clear messages between tests:

```go
t.Run("test name", func(t *testing.T) {
    // Clear at start
    client.DeleteAllMessages()
    
    // Your test...
    
    // Auto-cleanup with t.Cleanup
    t.Cleanup(func() {
        client.DeleteAllMessages()
    })
})
```

### 2. Reliable Waiting

Don't use fixed sleeps. Wait for conditions:

```go
// Bad
time.Sleep(1 * time.Second)

// Good
waitFor(t, func() bool {
    messages, _ := client.ListMessages(1, 10)
    return len(messages.Messages) > 0
}, 2*time.Second, 100*time.Millisecond)
```

### 3. Environment Configuration

Use environment variables for flexibility:

```go
func getSendriaURL() string {
    if url := os.Getenv("SENDRIA_URL"); url != "" {
        return url
    }
    return "http://localhost:1080"
}
```

### 4. Parallel Testing

Be careful with parallel tests - they can interfere:

```go
// If tests share Sendria instance, don't run in parallel
// t.Parallel() // AVOID

// Or use separate Sendria instances per test
```

### 5. Debugging Failed Tests

Save email content for debugging:

```go
if t.Failed() {
    // Dump all messages for debugging
    messages, _ := client.ListMessages(1, 100)
    for _, msg := range messages.Messages {
        t.Logf("Email: From=%s, To=%s, Subject=%s",
            msg.From[0].Email, msg.To[0].Email, msg.Subject)
        
        body, _ := client.GetMessagePlain(msg.ID)
        t.Logf("Body: %s", body)
    }
}
```

## Common Test Patterns

### Testing Email Verification Flow

```go
func TestUserRegistrationFlow(t *testing.T) {
    client := testhelpers.NewEmailTestClient(t)
    
    // 1. User registers
    err := YourApp.RegisterUser("newuser@example.com", "password123")
    if err != nil {
        t.Fatal(err)
    }
    
    // 2. Verify confirmation email sent
    msg := client.AssertEmailSent("newuser@example.com", "Confirm Your Email")
    
    // 3. Extract confirmation link
    body, _ := client.GetMessagePlain(msg.ID)
    linkRegex := regexp.MustCompile(`https://example\.com/confirm\?token=([a-zA-Z0-9]+)`)
    matches := linkRegex.FindStringSubmatch(body)
    if len(matches) != 2 {
        t.Fatal("Confirmation link not found")
    }
    token := matches[1]
    
    // 4. Confirm email
    err = YourApp.ConfirmEmail(token)
    if err != nil {
        t.Fatal(err)
    }
    
    // 5. Verify welcome email sent
    client.DeleteAllMessages() // Clear confirmation email
    client.AssertEmailSent("newuser@example.com", "Welcome to Our App!")
}
```

### Testing Rate Limiting

```go
func TestEmailRateLimiting(t *testing.T) {
    client := testhelpers.NewEmailTestClient(t)
    
    // Try to send many emails quickly
    for i := 0; i < 10; i++ {
        err := YourApp.SendNotification("user@example.com", "Test")
        if i < 5 {
            // First 5 should succeed
            if err != nil {
                t.Errorf("Email %d failed: %v", i+1, err)
            }
        } else {
            // Rest should be rate limited
            if err == nil || !strings.Contains(err.Error(), "rate limit") {
                t.Errorf("Email %d should have been rate limited", i+1)
            }
        }
    }
    
    // Verify only 5 emails sent
    messages, _ := client.ListMessages(1, 10)
    if len(messages.Messages) != 5 {
        t.Errorf("Expected 5 emails, got %d", len(messages.Messages))
    }
}
```

### Testing Email Templates

```go
func TestEmailTemplateVariables(t *testing.T) {
    client := testhelpers.NewEmailTestClient(t)
    
    user := User{
        Name:  "John Doe",
        Email: "john@example.com",
        Plan:  "Premium",
    }
    
    err := YourApp.SendAccountSummary(user)
    if err != nil {
        t.Fatal(err)
    }
    
    msg := client.AssertEmailSent(user.Email, "Your Account Summary")
    body, _ := client.GetMessagePlain(msg.ID)
    
    // Verify template variables replaced
    expectedTexts := []string{
        "Hi John Doe",
        "Plan: Premium",
        "Email: john@example.com",
    }
    
    for _, text := range expectedTexts {
        if !strings.Contains(body, text) {
            t.Errorf("Missing expected text: %q", text)
        }
    }
    
    // Verify no template variables left
    if strings.Contains(body, "{{") || strings.Contains(body, "}}") {
        t.Error("Unreplaced template variables found")
    }
}
```

## Troubleshooting

### Issue: EOF errors when running tests

**Solution**: Add connection pooling and read response bodies:

```go
client := &Client{
    httpClient: &http.Client{
        Transport: &http.Transport{
            MaxIdleConns:        10,
            MaxIdleConnsPerHost: 10,
            IdleConnTimeout:     90 * time.Second,
        },
    },
}
```

### Issue: Tests interfere with each other

**Solution**: Clear messages between tests and use unique subjects:

```go
subject := fmt.Sprintf("Test Email - %s - %d", t.Name(), time.Now().Unix())
```

### Issue: Emails not arriving in tests

**Solution**: Check Sendria is running and accessible:

```bash
curl http://localhost:1080/api/messages/
```

### Issue: HTML content doesn't match exactly

**Solution**: Sendria may normalize HTML. Test for content presence:

```go
// Instead of exact match
if html != expectedHTML { ... }

// Check contains key elements
if !strings.Contains(html, "<h1>Welcome</h1>") { ... }
```

## API Reference

### Client Methods

| Method | Description |
|--------|-------------|
| `NewClient(baseURL string, opts ...Option)` | Create a new client |
| `ListMessages(page, perPage int)` | List messages with pagination |
| `GetMessage(id string)` | Get full message details |
| `GetMessagePlain(id string)` | Get plain text content |
| `GetMessageHTML(id string)` | Get HTML content |
| `GetMessageSource(id string)` | Get raw email source |
| `GetMessageEML(id string)` | Download as EML file |
| `GetAttachment(messageID, cid string)` | Download attachment |
| `DeleteMessage(id string)` | Delete specific message |
| `DeleteAllMessages()` | Delete all messages |

### Options

```go
// With authentication
client := sendria.NewClient(url, sendria.WithBasicAuth("user", "pass"))

// With custom timeout
client := sendria.NewClient(url, sendria.WithTimeout(30*time.Second))
```

## Running Sendria

### Docker

```bash
docker run -p 1025:1025 -p 1080:1080 msztolcman/sendria
```

### Docker Compose

```yaml
version: '3.8'
services:
  sendria:
    image: msztolcman/sendria:latest
    ports:
      - "1025:1025"
      - "1080:1080"
    volumes:
      - sendria-data:/data
    environment:
      - SENDRIA_DB_PATH=/data/sendria.db

volumes:
  sendria-data:
```

### Python

```bash
pip install sendria
sendria --smtp-port 1025 --http-port 1080
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Acknowledgments

- [Sendria](https://github.com/msztolcman/sendria) - The SMTP server that makes this testing possible
- Built specifically for testing email functionality in Go applications