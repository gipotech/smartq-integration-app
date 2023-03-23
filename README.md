# SmartQ Integration Application
## Middleware for Cloud to third-party devices integration

### Docker quick start

Download a run the container
```shell
docker pull ghcr.io/gipotech/smartq-integration-app:latest
docker run --rm -it -p 8080:80 -v /configpath:/etc/smartq/config /logspath:/etc/smartq/logs gipotech/smartq-integration-app
```

Mounts:
- `/etc/smartq/config`: **[REQUIRED]** this is the mount point where the app will store its configuration and need to be persisted in order to preserve login and configuration between restarts
- `/etc/smartq/logs`: path where local logs files are stored

*Notes on Linux hosts: If you are hosting the docker container on linux and want to communicate with the host using `host.docker.internal`, you need to use `--add-host=host.docker.internal:host-gateway`. This is not needed on Windows or Mac. Docker 20.04+ is required.*

### Event schema
Events are forwarded as JSON and the schema is shared between all the forwarders (comments are added to clarify each field purpose).
```json
{
    "id": "56a5f02a-f757-4e1a-b960-79a4fba62f1a", // unique identifier for the event
    "queueId": "AMB1", // unique identifier of the queue
    "tenantId": "f23cd927-8638-4b9c-9110-8b47f8838f6e", // unique identifier of the tenant
    "identifier": "50", // ticket number
    "creatorUserId": "f23cd927-8638-4b9c-9110-8b47f8838f6e", // unique identifier of the creator of the event
    "enqueuedDateTime": "2022-01-01T09:53:11.4756174+01:00", // date-time in ISO-8601 format of when the ticket was added to the queue
    "managedDateTime": "2022-01-01T10:22:13.4786574+01:00", // date-time in ISO-8601 format of when the ticket was managed (called)
    "status": "Managed", // the changed status, can be: 'Enqueued', 'Managed', 'Closed' 
    "closeDateTime": null,  // date-time in ISO-8601 format of when the ticket was closed (customer is out of the queue)
    "description": "Custom description", // optional custom description of the appointment
    "room": "1", // optional custom description of the room used
    "patientIdentifier": "902365447", // unique identifier for the patient
    "appointmentId": "5479455", // unique identifier for the appointment
    "appointmentStartDateTime": "2022-01-01T09:30:00.0000000+01:00", // date-time in ISO-8601 format of when the appointment was scheduled to start
    "appointmentEndDateTime": "2022-01-01T09:45:00.0000000+01:00", // date-time in ISO-8601 format of when the appointment was scheduled to end
    "timestamp": "2022-01-01T10:22:13.4786574+00:00" // timestamp in ISO-8601 format of when the event occurred
}
```

### Forwarding options

#### Web hooks

The Web Hook forwarder will send all events received from SmartQ cloud service to a specific endpoint as JSON request.

*Configuration:*
- **Url**: the full URL of the endpoint that will receive events
- **Method**: HTTP method to use (default `POST`)
- **Headers**: list of optional headers to be added to the request (e.g. for authorization)
- **Allow invalid certificates**: enable this flag to allow the web hook forwarder to send messages also to endpoints using HTTPS but with invalid certificates (default disabled)

##### Sample:
- Url: http://host.docker.internal:8888/queue (make a request to the docker host)
- Method: POST
- Headers: Authorization: mytoken
```http request
POST http://host.docker.internal:8888/queue HTTP/1.1
Host: host.docker.internal:8888
Authorization: mytoken
Content-Type: application/json; charset=utf-8
Content-Length: 707

{
  "identifier": "100",
  "status": "Managing",
  // ... other properties omitted for brevity
}
```

#### RabbitMQ

The RabbitMQ forwarder will send all events received from SmartQ cloud service to a rabbitmq queue/topic as JSON serialized data.

*Configuration:*
- **Hostname**: hostname of the rabbitmq instance (default `localhost`)
- **Username**: username used to authenticate with rabbitmq (default `guest`)
- **Password**: username used to authenticate with rabbitmq (default `guest`)
- **Exchange name**: name of the exchange used to route messages in rabbitmq (default none)
- **Queue name**: name of the queue where messages are forwarded in rabbitmq (default `smartq`)
- **Port**: rabbitmq port to use (default `-1`, which is "automatic")

Events are enqueued as JSON using the common schema

#### Microsoft SQL Server

The Microsoft SQL Server forwarder will insert all events data received from SmartQ cloud service to a table in a SQL Server database (metadata + JSON body).

*Configuration:*
- **Server name**: server host name and port + instance (e.g. localhost)
- **Database name**: name of the database (created automatically if it does not exist)
- **Username**: username of MSSQL server user
- **Password**: password of MSSQL server user
- **Encrypt**: enable if the MSSQL server instance will use encryption (default: YES)
- **Trust server certificate**: enable to bypass certificate validation of the server (default: NO)
- **Connection timeout**: timeout of the connection phase (default 30 seconds)
- **Table name**: name of the table to create and use for storing events (default `SmartQEvents`)

The table schema is the following:
```sql
CREATE TABLE {TableName}
(
    [EventId] INT NOT NULL AUTOINCREMENT PRIMARY KEY,
    [QueueId] NVARCHAR(255) NOT NULL,
    [Identifier] NVARCHAR(100) NOT NULL,
    [Status] NVARCHAR(30) NOT NULL,
    [Timestamp] DATETIME2 NOT NULL,
    [EventBody] NVARCHAR(MAX) NOT NULL
)
```
Events are inserted in the table and serialized as JSON (using the common schema) inside `EventBody` column.
