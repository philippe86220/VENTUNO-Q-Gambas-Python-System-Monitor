# System Monitor — Gambas 3 and Python HTTP Server

A small Gambas 3 application that displays Linux system information retrieved from a local HTTP server written in Python.

The application displays three measurements:

- CPU temperature
- RAM usage
- System load average

This project demonstrates communication between a Python HTTP server and a Gambas 3 graphical application using HTTP and JSON.

The project was developed and tested on an **Arduino VENTUNO Q running Ubuntu Linux**.

## Architecture

```text
┌─────────────────────┐   HTTP GET /mesures   ┌─────────────────────┐
│                     │ ────────────────────► │                     │
│    Gambas 3 GUI     │                       │    Python Server    │
│    (infos_systeme)  │ ◄──────────────────── │    (moniteur.py)    │
│                     │      JSON response    │                     │
└─────────────────────┘                       └──────────┬──────────┘
                                                       │
                                                       ▼
                                             Linux system information
                                             ├── /proc/meminfo
                                             ├── /sys/class/thermal
                                             └── os.getloadavg()
```

The Python server reads information directly from the Linux operating system.

The Gambas application sends an HTTP request, receives a JSON response, decodes it, and displays the values.

No Arduino App Lab application, Docker container, or microcontroller is required for this version.

## Requirements

### Operating system

Linux with access to:

- `/proc/meminfo`
- `/sys/class/thermal`

### Python

Python 3 (tested with Python 3.12).

The server uses only Python standard library modules. No additional Python packages are required.

### Gambas

Gambas 3 with the following components enabled:

- `gb.net`
- `gb.net.curl` — HTTP client support
- `gb.util.web` — JSON decoding

On Ubuntu or Debian, the HTTP component can be installed with:

```bash
sudo apt install gambas3-gb-net-curl
```

## Python HTTP Server

### Starting the server

Open a terminal and run:

```bash
python3 moniteur.py
```

The server listens on:

```text
127.0.0.1:8000
```

The loopback address `127.0.0.1` restricts access to applications running on the same Linux system.

### HTTP endpoint

| Method | Endpoint | Response |
|---|---|---|
| GET | `/mesures` | JSON, HTTP 200 |
| GET | Any other path | HTTP 404 |

Full endpoint:

```text
http://127.0.0.1:8000/mesures
```

### Example JSON response

```json
{
  "temperature": 36.4,
  "ram": 29.1,
  "charge": 0.79
}
```

### Measurements

| JSON key | Description | Unit |
|---|---|---|
| `temperature` | Highest valid temperature among CPU thermal zones | °C |
| `ram` | Used RAM percentage calculated from `MemTotal` and `MemAvailable` | % |
| `charge` | System load average over the last minute | None |

#### CPU temperature

The server reads thermal zones under:

```text
/sys/class/thermal/
```

It selects zones whose type starts with `cpu-` and returns the highest valid temperature.

If no valid CPU temperature is available, the JSON value is `null`.

#### RAM usage

RAM usage is calculated using values from:

```text
/proc/meminfo
```

The calculation is:

```text
RAM usage (%) = (MemTotal - MemAvailable) / MemTotal × 100
```

#### System load

The server uses:

```python
os.getloadavg()[0]
```

This returns the system load average over one minute.

**Note:** The load average is not a CPU usage percentage. It represents the average system workload and must be interpreted in relation to the number of available CPUs.

### Testing the server

Open another terminal and execute:

```bash
curl -i http://127.0.0.1:8000/mesures
```

A successful response should contain:

```text
HTTP/1.0 200 OK
```

followed by the JSON data.

## Gambas 3 Client

### Creating the project

1. Open Gambas 3.
2. Create a graphical application named `infos_systeme`.
3. Create or open the main form `FMain`.
4. Enable the following components in **Project → Properties → Components**:
   - `gb.net`
   - `gb.net.curl`
   - `gb.util.web`
5. Add two controls to the form:
   - `Button1` — requests system information.
   - `TextArea1` — displays the measurements.

### Gambas code

Add the following code to the button's click event:

```gambas
Public Sub Button1_Click()

  Dim hClient As HttpClient
  Dim sReponse As String
  Dim cData As Collection

  hClient = New HttpClient
  hClient.URL = "http://127.0.0.1:8000/mesures"
  hClient.Async = False
  hClient.Timeout = 10

  hClient.Get()

  If Lof(hClient) = 0 Then
    Message.Error("No response from server")
    Return
  Endif

  sReponse = Read #hClient, Lof(hClient)
  cData = JSON.Decode(sReponse)

  TextArea1.Text = "CPU temperature: " & cData["temperature"] & " °C" & gb.NewLine
  TextArea1.Text &= "RAM usage: " & cData["ram"] & " %" & gb.NewLine
  TextArea1.Text &= "System load: " & cData["charge"]

End
```

### How the Gambas client works

**1. Create an HTTP client**

```gambas
hClient = New HttpClient
```

The `HttpClient` class is provided by the `gb.net.curl` component.

**2. Configure the server address**

```gambas
hClient.URL = "http://127.0.0.1:8000/mesures"
```

**3. Send the HTTP request**

```gambas
hClient.Get()
```

With `Async = False`, the request is synchronous.

The application waits for the HTTP operation to complete before continuing.

**4. Read the response**

```gambas
sReponse = Read #hClient, Lof(hClient)
```

The HTTP response body is stored in a Gambas string.

**5. Decode the JSON**

```gambas
cData = JSON.Decode(sReponse)
```

The JSON object is converted into a Gambas `Collection`.

Individual values can then be accessed using their keys:

```gambas
cData["temperature"]
cData["ram"]
cData["charge"]
```

**6. Display the measurements**

The values are displayed in `TextArea1`.

## Running the Application

1. Start the Python server:

```bash
python3 moniteur.py
```

2. Open the Gambas project.

3. Run the Gambas application using **F5**.

4. Click the button to request the measurements.

Example output:

```text
CPU temperature: 36.4 °C
RAM usage: 29.1 %
System load: 0.79
```

The displayed values depend on the current system state.

## Troubleshooting

| Problem | Possible cause | Solution |
|---|---|---|
| `Unknown identifier: HttpClient` | Missing Gambas component | Enable `gb.net.curl` |
| `Address already in use` | Port 8000 is already occupied | Check with `ss -ltnp` |
| No response from server | Python server is not running | Start `moniteur.py` |
| Empty HTTP response | Response was not read correctly | Use `Read #hClient, Lof(hClient)` |
| Missing CPU temperature | No matching CPU thermal zone | Inspect `/sys/class/thermal/thermal_zone*/type` |
| JSON decoding error | Invalid or unexpected response | Inspect the HTTP response |
| GTK warnings | Graphical environment warning | Check whether the application still operates normally |

### Checking port 8000

```bash
ss -ltnp | grep ':8000'
```

### Checking CPU thermal zones

```bash
cat /sys/class/thermal/thermal_zone*/type
```

## Possible Improvements

- Add a Gambas `Timer` to refresh measurements automatically.
- Handle unavailable temperature values (`null`).
- Display additional system information, such as disk usage or CPU frequency.
- Improve HTTP error handling and JSON validation.
- Use asynchronous HTTP requests for remote servers.
- Move the Python server into an Arduino App Lab custom brick.
- Allow a native Gambas application to communicate with a service running inside a Docker container.

## Future Development

The next step is to integrate the Python monitoring server into an **Arduino App Lab custom brick**.

The objective is to understand how a Docker container can access hardware and operating-system information from the Linux host.

A subsequent step will demonstrate communication between this custom brick and the Gambas graphical application.

These experiments will focus exclusively on the VENTUNO Q's Linux MPU.

No MCU communication is required.

## License

Add the license of your choice before distributing or reusing this project.
