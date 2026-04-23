# 🚀 Service Bus Local Setup Guide (Docker + ZIP Package)

This guide walks you through setting up a **local Azure Service Bus Emulator** using **Docker** and the provided ZIP package.

---

## 📦 Prerequisites

### 1. Install Docker Desktop

Download and install Docker:

👉 https://www.docker.com/products/docker-desktop

**Steps:**
1. Install Docker Desktop
2. Enable:
   - ✅ WSL 2 (recommended)
3. Restart your machine if required

Verify installation:

```bash
docker --version
docker compose version
````

---

## 📁 Project Setup

### 2. Extract the ZIP File

Extract the provided ZIP file to:

```bash
E:\ServiceBus
```

Expected structure:

```text
ServiceBus/
│
├── docker-compose.yml
├── run.bat
└── README.md
```

---

## ⚙️ Configuration

Ensure your `docker-compose.yml` is properly configured.

### Example Configuration

```yaml
version: '3.8'

services:
  servicebus:
    image: mcr.microsoft.com/azure-messaging/servicebus-emulator
    container_name: servicebus-local
    restart: always
    ports:
      - "5672:5672"   # AMQP TCP
      - "9354:9354"   # Service Bus TCP
    environment:
      - ACCEPT_EULA=Y
    volumes:
      - ./data:/data
```

---

## ▶️ Running the Service

### Option 1: Using `run.bat`

```bat
run.bat
```

### Option 2: Manual Docker Command

```bash
docker compose up -d
```

To stop:

```bash
docker compose down
```

---

## 🌐 Endpoints

| Protocol   | Endpoint                |
| ---------- | ----------------------- |
| AMQP (TCP) | `amqp://localhost:5672` |
| TCP        | `localhost:9354`        |

---

## 🔑 API Setup for Service Bus Configuration

Add the following configuration to your `appsettings.json` or `appsettings.Development.json`:

```json
{
  "ConnectionStrings": {
    "ServiceBus": "Endpoint=sb://localhost;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true;EntityPath=fel-events"
  },
  "ServiceBusTopicName": "fel-events",
  "ServiceBusSubscriptionName": "events"
}
```

---

## ⚡ Azure Functions / ServiceBusTrigger Setup

For projects that use `ServiceBusTrigger`, add the following to `local.settings.json`:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "ServiceBusConnection": "Endpoint=sb://localhost;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true;EntityPath=fel-events"
  }
}
```

### Example Trigger

```csharp
[Function("ProcessFelEvents")]
public async Task Run(
    [ServiceBusTrigger("fel-events", "events", Connection = "ServiceBusConnection")]
    ServiceBusReceivedMessage message)
{
    string body = message.Body.ToString();
}
```

---

## ⚠️ IMPORTANT: Transport Configuration (Required for Local Emulator)

The **local Service Bus Emulator does NOT support `AmqpWebSockets`**.

You **must change the transport type to `AmqpTcp`**.

### ❌ Default (Cloud - NOT working locally)

```csharp
TransportType = ServiceBusTransportType.AmqpWebSockets;
```

### ✅ Correct (Local Emulator)

```csharp
var serviceBusClient = new ServiceBusClient(
    serviceBusConnectionString,
    new ServiceBusClientOptions
    {
        RetryOptions = new ServiceBusRetryOptions
        {
            Mode = ServiceBusRetryMode.Exponential,
            MaxRetries = 5,
            Delay = TimeSpan.FromSeconds(2),
            MaxDelay = TimeSpan.FromSeconds(60),
            TryTimeout = TimeSpan.FromMinutes(1),
        },
        TransportType = ServiceBusTransportType.AmqpTcp, // ✅ REQUIRED for local
    });
```

### 📌 Why this is required

* `AmqpWebSockets` → used for Azure cloud (port 443)
* `AmqpTcp` → required for local emulator (port 5672)

---

## 💻 .NET Usage

```csharp
var client = new ServiceBusClient(serviceBusConnectionString);
var sender = client.CreateSender("fel-events");
```

---

## 📂 Data Persistence

```text
./data
```

* Stores messages locally
* Keeps logs for debugging

---

## 🧪 Verify Setup

```bash
docker ps
```

Expected:

```text
servicebus-local
```

---

## 🛠️ Useful Commands

```bash
docker compose restart
docker logs servicebus-local
```

---

## ❗ Troubleshooting

### Cannot Connect

* Ensure Docker is running
* Ensure ports `5672` / `9354` are not used
* Ensure `AmqpTcp` is configured
* Check logs:

```bash
docker logs servicebus-local
```

---

### Invalid EntityPath

Make sure this matches:

```json
"EntityPath=fel-events"
```

and:

```json
"ServiceBusTopicName": "fel-events"
```

---

## ⚠️ Notes

* Local emulator is **not full Azure Service Bus**
* Some advanced features may not work
* Intended for **development/testing only**

---

## ✅ Summary

✔ Install Docker
✔ Extract ZIP
✔ Run container
✔ Configure API + Azure Functions
✔ Use `AmqpTcp` (NOT WebSockets)
✔ Use topic `fel-events` + subscription `events`
✔ Ready for local development
