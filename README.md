# Guenther - a Tool to Test for Server Side Request Abuse

Here is a quick tutorial on guenther. For more information, please refer to our paper http://trouge.net/gp/papers/SSR_raid2016.pdf

## Quick recap on Server Side Request

Server Side Requests SSR is a communication pattern in which an intermediate server can retrieve an HTTP resource on behalf of a user. Guenther tests intermediate SSR servers to detect vulnerabilities that can be used to mount SSR-based attacks.

## Guenther Components

Guenther has the following components:

* **Tester** the goal of the tester is to probe SSR server with a number of messages which are meant for the Monitor (see below). In each test, the Tester prepares a message containing a URL of the Monitor. The position of the URL in the message is specified via a JSON-based input file.

* **Monitor** is a server offering a number of services to support the testing activity. The main services are the *detection service* and the *HTTP redirector*. The detection service plays the role of an external service hosting a resource. It waits for an incoming message generated by the SSR server. The HTTP redirector service is used to test against input filtering bypass. When the redirector is used, the tester generates a message which does not contain the URL of the detection service. Instead, it will contain the URL of the redirection service. Then, the redirection service redirects the SSR server towards the detection service. The monitor hosts other services. For example, an FTP daemon, a TCP server, and a web service to test the JS capabilities of user agents.

# Examples

## Simple one

This example tests an SSR service to check whether it supports the generation of HTTP request towards a target `monitor.foo.com`. The generation of input file is described below.

```bash
$ guenther.py webapp_A.json -tD -l monitor.foo.com
```

`-tD` performs a test in which the URL to be sent to the SSR contains the FQDN of the detection service, i.e., `monitor.foo.com`. The parameter `-l` can be a hostname or an IP that the monitor will listen to for incoming messages. In case guenther is executed from a local network behind a NAT/MASQUERADE, use the following parameters:

```bash
$ guenther.py webapp_A.json -tD -l 10.0.0.1 -p monitor.foo.com
```

where 10.0.0.1 is the local IP and `monitor.foo.com` is the public IP of
the gateway router. This example will work if there are in place port forwarding rules
between 10.0.0.1 and `monitor.foo.com`. Both `-p` and `-l` accept IPs and hostnames.

## Example 2: Bypassing FDQN only filters

Now, let's assume that the SSR service accepts messages with FDQN URLs. This can be
discovered with the previous command plus the following command:

```bash
$ guenther.py webapp_A.json -tI -l monitor.foo.com
```

The output of this command will show that no message from the SSR service has been received
by the detection service. However, it is still possible to bypass this filter
with the aid of the HTTP redirector:

```bash
$ guenther.py webapp_A.json -tI -tRx -l monitor.foo.com
```

The parameter `-tRx` will change the behavior of `-tI` in which
the URL with the IP of the detection service is sent via redirection and not
directly.

The last two tests can be executed at once as follows:

```bash
$ guenther.py webapp_A.json -tI -tR -l monitor.foo.com
```

As opposed to `-tRx`, `-tR` performs two tests: with and without
HTTP redirection.

# Input File Format

The input file of guenther is a JSON object stored in a file provided via command line. 

```json
{
  "urlp"   : $s$,
  "queryp" : {
              $p_1$: $v_1$,
              $p_2$: $v_2$,
              ...
              $p_n: $v_n,
               }, 
  "headers": {
              $hn_1$: $hv_1$,
              $hn_2$: $hv_2$,
              ...
              $hn_n: $hv_n,
             }, 
  "bodyp"  : {
              $p'_1$: $v'_1$,
              $p'_2$: $v'_2$,
              ...
              $p'_n: $v'_n,
             }, 
  "method" : $m$
}
```

* **method** (mandatory) is a string $s$ for the HTTP request method name, i.e., `"GET"`, `"POST"`.

* **urlp** (mandatory) is string which contains an URL of the SSR service without query string.
   
* **queryp** (optional) is a dictionary which maps a parameter $p_i$ with a value $v_i$ where $i=1, ..., n$. Values can also be *placeholders*. Guenther supports only the placeholder string `"{monitor}"` that is replaced at runtime either with the URL of the detection service or with the URL of the redirection service. 

* **headers** (optional) is a dictionary which maps header names $hn _i$, e.g., `User-Agent`, with header values $hv_i$, e.g., `Mozilla/5.0 (X11; Ubuntu; Linux x86\_64; rv:33.0) Gecko/20100101`.
   
* **bodyp** (optional) is a dictionary as seen for **queryp**.

## Example of Input File

Let's assume that the one below is the HTTP request to submit a URL to the SSR service:

```
POST /send HTTP/1.1\r\n
Host: localhost:8080\r\n
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:33.0) Gecko/20100101 Firefox/33.0\r\n
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8\r\n
Accept-Language: en-US,en;q=0.5\r\n
Accept-Encoding: gzip, deflate\r\n
Referer: http://localhost:8080/\r\n
Connection: keep-alive\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Content-Length: 23\r\n
\r\n
your_url=http://monitor.guenter.corp/\r\n
```

The below JSON object input file is the following:
```json
{
  "urlp"   : "http://localhost:8080/send",
  "headers": {
              "Content-Type": "application/x-www-form-urlencoded"
             }, 
  "queryp" : {}, 
  "bodyp"  : {
               "your_url": "{monitor}"
             }, 
  "method" : "POST"
}
```

## `raw2gnt.py`: Utility to generate input files from RAW HTTP Request

 You can create JSON object input files starting from raw HTTP requests stored in a file with `utils/raw2gnt.py`:

```bash
  $ cat example/rawreq.txt | python src/utils/raw2gnt.py > GNT_file.json
```

Then, modify the json file and add the placeholder `{monitor}`. If the request needs to be sent over HTTPS, use the `-https` flag:

```bash
  $ cat example/rawreq.txt | python src/utils/raw2gnt.py -https > GNT_file.json
```


## Install

See `requirements.txt`.