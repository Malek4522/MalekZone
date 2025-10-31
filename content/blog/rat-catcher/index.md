+++
title = "Catching Remote Administration Trojans (RATs)"
description = "Session-aware detection of RAT protocols with RAT Catcher."
authors = ["MALEK452"]
date = 2025-10-30
[taxonomies]
tags = ["Security", "Malware", "RATs"]
+++
<p>
    
  </p>
  <p>
  <i>
   This blog post is based on the research paper "Catching Remote Administration Trojans (RATs)". For full methodology and data, please see the original publication.
   </i>
  
  <p>
    <img src="../../rat-catcher/RAT%20Catcher.png" alt="RAT Catcher image" style="max-width: 100%; height: auto; border-radius: 6px;" />
  </p>
  <h2>
   What Is a RAT?
  </h2>
<b>A Remote Administration Trojan (RAT)</b> is a stealthy, trojanized backdoor that grants an attacker near-complete remote control of a compromised machine. Typically the RAT installs a hidden server component on the victim that listens on a TCP or UDP port, while the attacker runs a client interface to connect and issue commands — effectively allowing the attacker to operate the target “as if sitting at the keyboard.” This covert client-server arrangement means the adversary does not need credentials or physical access; they simply communicate over the network to control the host.

Once installed, RATs expose a broad attack surface: they can capture keystrokes and screenshots, record audio/video, intercept passwords and clipboard contents, and exfiltrate files. Attackers can also manipulate the system — create, delete, or transfer files; spawn or kill processes; edit the registry; and reconfigure or disable security tools such as antivirus or firewalls. RATs may also route or sniff network traffic, turn the host into a proxy for further attacks, or enroll it in a botnet used for DDoS or data storage.

Modern RATs emphasize persistence and stealth. They frequently piggyback on legitimate programs (using binders or host-patching) to auto-start on reboot, and they use anti-analysis and anti-detection techniques to evade host-based scanners and EDR. Their network layer is often polymorphic: custom protocols, encryption, and dynamic TCP/UDP ports are used so RAT traffic resembles benign services like HTTP or FTP and avoids simple signature detection. Together these features make RATs powerful, flexible, and hard to detect.
  <p>
    <b>RAT distribution:</b> Attackers typically deliver RAT servers concealed inside seemingly benign software. Before deployment, RAT binaries are pre-configured with tools called <b>binders</b> (for example, "<b>EditServer</b>" for SubSeven or "<b>bo2kcfg</b>" for Back Orifice 2000) that set parameters like <b>default ports</b>, <b>encryption settings</b>, and <b>default passwords</b>. The resulting <b>RAT executable</b> is often renamed or bundled as a <b>software patch</b>, <b>game</b>, or <b>email attachment</b>. When the user runs this <b>dropper</b>, the RAT server installs itself – sometimes <b>piggybacking</b> on a legitimate process (e.g. as a thread inside <b>Explorer</b> or <b>Internet Explorer</b>) so that it starts <b>automatically</b> and <b>invisibly</b> on each <b>boot</b>.
  </p>
  <p>
    <b>Feedback channels:</b> Some RAT clients include features to report back the victim's <b>IP</b> and <b>listening port</b> to the attacker. For instance, a RAT might connect out over <b>IRC</b>, <b>email</b>, or <b>P2P networks</b> so that the attacker learns how to reach the hidden server. In this way, even if the victim is behind a <b>NAT</b> or <b>firewall</b>, the attacker can eventually establish <b>connectivity</b> using the reported address.
  </p>
  <p>
    <b>Persistence:</b> Once installed, RATs often make <b>registry edits</b>, modify <b>startup scripts</b>, or drop files in <b>system folders</b> so that they remain active across <b>reboots</b>. Because they blend into normal processes, conventional host-based scanners may miss them. As the authors note, this "<b>bootstrapping</b>" behavior (e.g. editing <b>win.ini</b> or adding <b>startup entries</b>) means <b>network-based detection</b> (monitoring the machine's traffic) is often more effective than simply watching for new files or processes 
  </p>
  </p>
  <p>
   <b>
    RAT Functionalities and Examples:
   </b>
   Typical RAT commands (collected from SubSeven and Back Orifice) illustrate their capabilities. For example, SubSeven supports commands like
   <code>
    FFN
   </code>
   (file find),
   <code>
    IRG
   </code>
   (registry editor), and
   <code>
    TKS
   </code>
   (keystroke logger), while Back Orifice provides commands such as
   <code>
    KEYLOG
   </code>
   (keystroke logging) and
   <code>
    PROCKILL
   </code>
   (kill a process)
   . The table below shows a few representative commands from these RATs:
  </p>
  <table>
   <thead>
    <tr>
     <th>
      SubSeven
      <br>
      Command (ID)
     </th>
     <th>
      Description
     </th>
     <th>
      Back Orifice
      <br>
      Command (ID)
     </th>
     <th>
      Description
     </th>
    </tr>
   </thead>
   <tbody>
    <tr>
     <td>
      GMI (1)
     </td>
     <td>
      Get system information
     </td>
     <td>
      PING (01)
     </td>
     <td>
      Ping the current host
     </td>
    </tr>
    <tr>
     <td>
      FFN (7)
     </td>
     <td>
      Find files/folders
     </td>
     <td>
      KEYLOG (07)
     </td>
     <td>
      Record keystrokes to file
     </td>
    </tr>
    <tr>
     <td>
      IRG (13)
     </td>
     <td>
      Registry editor (add/
      <br>
      remove keys)
     </td>
     <td>
      PROCKILL (20)
     </td>
     <td>
      Kill a process
     </td>
    </tr>
    <tr>
     <td>
      TKS (8)
     </td>
     <td>
      Key logger (record
      <br>
      keystrokes)
     </td>
     <td>
      APPADD (0E)
     </td>
     <td>
      Spawn a console
      <br>
      application
     </td>
    </tr>
   </tbody>
  </table>
  
  <p>
   These commands highlight how RATs
   <i>
    act like remote shells
   </i>
   : they can browse the file system, dump passwords or registry entries, and capture live data (e.g. video or audio). For instance, an attacker using the
   <b>
    Eclipse RAT
   </b>
   (an FTP-based Trojan) may see a login banner and interact with it exactly like an FTP server
   . In this sample session, the victim (RAT server) presents an FTP banner and accepts a
   <code>
    USER/PASS
   </code>
   login, then processes
   <code>
    NLST
   </code>
   to open a data channel for directory listing (all while actually serving attacker commands):
  </p>
  <pre>Victim:3791 → Attacker:1074 : 220 EclYpse's FTP Server is happy to serve you
(FTP banner)
Attacker:1074 → Victim:3791 : USER (none) (attacker attempts login)
Victim:3791 → Attacker:1074 : 331 Password required (server prompts)
Attacker:1074 → Victim:3791 : PASS xxxxxx (attacker sends password)
Victim:3791 → Attacker:1074 : 230 User (none) logged in (login success)
Attacker:1074 → Victim:3791 : PORT 192,168,5,143,4,51 (setup data channel)
Victim:3791 → Attacker:1074 : 200 Port command successful
Attacker:1074 → Victim:3791 : NLST (request file listing)
Victim:1030 → Attacker:1075 : (SYN) (data channel established)</pre>
  <p>
   The above stream shows how a Trojan can
   <b>
    camouflage
   </b>
   its traffic as legitimate FTP. EclYpse dynamically opens a data channel on a different port, much like FTP, making it hard for a simple signature or port-based filter to notice
   . Other RATs adopt similar tricks: they can tunnel commands over HTTP or Telnet-like sessions, hop ports, or pad payloads with random data to foil pattern matching
   . In summary, RAT communications are
   <b>
    session-oriented
   </b>
   and often encrypted or obfuscated, which defeats static signature checks and simple protocol filters.
  </p>
  <h2>RAT Communication and Evasion Techniques</h2>
  <p>
    RATs are notorious for evasion tricks to defeat signature-based detectors. Two key strategies are <b>encryption</b> and <b>protocol camouflage</b>:
  </p>
  <p>
    <b>Encrypted Traffic:</b> Many RATs (like <i>Back Orifice</i>, <i>BO2K</i>, and <i>NetBus</i>) XOR-encrypt their commands and data using a key derived from the attacker’s chosen password. This scrambling thwarts simple payload signatures. However, <b>XOR</b> is a weak cipher: it does not change message lengths or fully obfuscate patterns. For example, every <i>Back Orifice</i> packet still begins with the magic string <code>*!*QWTY?</code> once decrypted. In practice, <b>RAT Catcher</b> can brute-force the 4-byte key and recover cleartext, then detect the structured commands inside (e.g. <code>PING</code>, <code>PROCESSLIST</code>, <code>INFO</code>). The lesson: encryption hides content only superficially — a smart detector can reconstruct and inspect the protocol semantics to expose the RAT.
  </p>
  <p>
    <b>Protocol Diversification:</b> Modern RATs use a variety of transport protocols and mimic normal services. Some are <b>TCP</b>-based (<i>NetBus</i>, <i>SubSeven</i>), others <b>UDP</b>-based (original <i>Back Orifice</i>), and many can do both. For example, <i>Back Orifice 2K</i> supports <b>TCP</b>, whereas older <i>BO</i> only used <b>UDP</b>. Even more devious, RATs often impersonate legitimate protocols. <i>Eclypse</i> masquerades as an <b>FTP</b> server — it starts with an FTP-style banner like <code>220 Eclypse’s FTP server is happy to see you!</code>. <i>WanRemote</i> embeds its commands in <b>HTTP</b> GET requests (e.g. <code>GET /fm?cd=C:/ HTTP/1.1</code> to change directories). <i>Drat</i> and other Telnet-like RATs provide a live shell, echoing each character typed by the attacker. Because of this, RAT traffic can look almost indistinguishable from benign <b>FTP</b>, <b>HTTP</b>, or <b>Telnet</b> sessions.
  </p>
  <p>
    <b>Other Evasion:</b> RATs generate noise and misdirection. Some insert random “decoy” packets or text to trigger false alarms (cgi.di.uoa.gr). For instance, the <i>Doly</i> RAT routinely attempts connections on many ports (3456, 4567, …) which looks like scanning but is in fact a way to camouflage its real channel (cgi.di.uoa.gr). Attackers also constantly change ports (“<b>port hopping</b>”) so that no single fixed port signature works (cgi.di.uoa.gr). These tricks force defenders to analyze content rather than rely on fixed signatures or ports.
  </p>
  <h2>
   The RAT Catcher Framework
  </h2>
  <p>
   Given these challenges, Chen <i>et al.</i> propose <b>RAT Catcher</b>, a network-based system that examines every connection at the application layer. It works <b>inline</b>, reassembling and correlating traffic streams to spot RAT protocols even when they hide among normal data
   . <b>RAT Catcher</b> has four main components:
  </p>
  <ul>
   <li>
    <b>Session Correlator</b>: This module tracks all active connections. Each new packet is placed into a session keyed by (source IP, source port, dest IP, dest port, protocol)
    . Session records may include control channels and any data channels (even if the RAT uses multiple streams). By remembering past sessions, the correlator can link together related connections (e.g. a data-transfer channel spawned by a known control session)
    . Essentially, the correlator ensures that every packet is associated with the right context so RAT activity can be spotted over the entire conversation.
</li>
<li>

<b>Message Sequencer</b>: Once packets are associated with a session, this module reassembles them into complete application messages. It manages TCP stream reordering by tracking sequence numbers and buffering out-of-order segments in an interval tree. For UDP, it groups packets by session. The goal is to recover the original application-layer message boundaries so that a multi-packet RAT command is seen as one coherent message; without this reassembly, a single command split over two TCP segments might slip through unnoticed.
</li>
   <li>
    <b>Traffic Distinguisher</b>: Now in possession of whole messages, RAT Catcher classifies the session's protocol. A <b>Traffic Classifier</b> first sorts traffic into general categories using simple heuristics
    . It looks for telltale patterns (e.g. "GET ... HTTP/1.1" for <b>HTTP</b>, or common <b>FTP</b> commands). Based on this, each session is handed to a specialized dissector for <b>HTTP</b>, <b>FTP</b>, <b>Telnet</b>, or <b>Miscellanea</b>. The <b>Miscellanea</b> dissector contains protocol-specific analyzers for proprietary RAT protocols. For example, the HTTP dissector can parse Eclipse's FTP-style responses by scanning for the "Eclipse" banner, and the Telnet dissector watches for the character-echo behavior of Drat
    . In short, the Traffic Distinguisher uses a multi-stage approach: first sort by common protocol, then apply targeted checks for known RAT message formats.
   </li>
   <li> 

  <b>Trojan Terminator</b>: When a session is identified as a RAT (its "<b>TYPE</b>" and "<b>CONFIRM</b>" flags are set by the above analyses), the Terminator takes action. It can generate alerts or logs and drop/block further packets from that connection
    . It can even <b>hijack the session</b>: RAT Catcher can pretend to be the RAT server and talk to the attacker, effectively turning the RAT channel into a honeypot
    . In more aggressive mode, the Terminator can exploit the RAT's own cleanup commands. For instance, NetSphere's client has a "<code>&lt;KillServer&gt;</code>" command that causes the server to shut down and remove itself. RAT Catcher can simulate this command on the victim's machine to force the RAT to uninstall
    . Similarly, other RATs like GateCrasher have "<b>uninstall</b>" routines that the Terminator can trigger. This proactive step can neutralize the malware entirely.
   </li>
  </ul>
  <p>
   Together, these modules allow <b>RAT Catcher</b> to correlate disparate clues (session tuples, packet contents, protocol behaviors) and reliably catch RAT traffic even under obfuscation.
  </p>
  <h2>
   Evaluation Results
  </h2>
  <p>
   In lab tests, RAT Catcher proved extremely effective. The authors implemented it in C on a FortiGate-300 IPS and generated numerous RAT sessions (including
   <i>
    Back Orifice
   </i>
   ,
   <i>
    NetBus
   </i>
   ,
   <i>
    SubSeven
   </i>
   ,
   <i>
    DeepThroat
   </i>
   ,
   <i>
    HellDriver
   </i>
   , etc.) on mixed Windows/Linux hosts. In the initial manual tests, RAT Catcher correctly flagged
   <b>
    all
   </b>
   malicious sessions across multiple coexisting RATs.
    In automated stress tests, a RAT emulator replayed tens of thousands of synthetic RAT sessions: RAT Catcher maintained
   <b>
    100% detection
   </b>
   in almost every scenario, while a Snort IDS (using published RAT rules) often failed catastrophically. For instance, when flooding the analyzer with NetBus traffic, RAT Catcher “creates no false negatives” and catches every session, whereas Snort missed the majority
   . This robustness comes from RAT Catcher’s message-level analysis: it treats variable-length fields and full byte sequences flexibly (so, for example, it catches NetBus even when messages aren’t the exact size Snort expects).
  </p>
  <p>
   Scalability tests showed RAT Catcher scales to very high loads. Using 20 tester machines, the authors replayed up to 700,000 RAT sessions (with up to 50,000 concurrent benign FTP sessions as background noise)
   . The RAT Catcher still labeled
   <b>
    all sessions
   </b>
   correctly, with no false positives – in almost all cases it stayed at 100% catch rate. Only under extreme synthetic delays did a tiny fraction of detections drop (worst case 99.80% detected).

   In other words, RAT Catcher lost only 0.2% of RAT sessions under the absolute heaviest (and unrealistic) stress test. Even then, only the
   <i>
    second
   </i>
   confirmation alert was missed; the first-alert “tentative RAT” marking still occurred
   . Importantly, this processing was inline in a 400 Mb/s IPS – the RAT Catcher added minimal latency, consistent with the expected 1–2 second “think time” of a human operator using a RAT client.
  </p>
  <p>
   Finally, RAT Catcher was tested
   <b>
    in the real world
   </b>
   . FortiGate appliances with RAT Catcher were deployed at universities in France, China, and the USA for two weeks
   . They forwarded all traffic to a Threat Analysis Center (TAC) for verification. Across thousands of connections, RAT Catcher detected 100% of observed
   <i>
    Back Orifice
   </i>
   ,
   <i>
    NetBus
   </i>
   , and
   <i>
    SubSeven
   </i>
   instances. In fact, every confirmed RAT session at TAC had been caught by RAT Catcher, with no missed cases
   . (By comparison, Snort missed many Trojan sessions in these traces, especially for NetBus and SubSeven, due to its rigid rule patterns
   .) In short, RAT Catcher delivered
   <i>
    perfect or near-perfect
   </i>
   detection in both controlled and live traffic with negligible false positives
   .
  </p>
  <h2>
   Conclusion and Future Directions
  </h2>
  <p>
   RAT Catcher’s key insight is
   <b>
    session-aware, grammar-based analysis
   </b>
   . Instead of static string matching or fixed port checks, it tracks each TCP/UDP connection from start to finish, reassembles all application data,
  </p>
  <p>
   and applies knowledge of RAT protocol syntax. This allows it to defeat port-hopping, encryption (as long as decipherable), and payload obfuscation. In the authors' words, "The RAT Catcher inspects every passing packet and maintains information for the entire lifetime of sessions"
   , dissecting both directions to ensure conformance with the Trojan's protocol. The result is that
   <i>
    no RAT connection slips through unnoticed
   </i>
   , and legitimate traffic is left intact
   .
  </p>
  <p>
   As a proactive defense, RAT Catcher can even take corrective action on-the-fly. Upon confirming a RAT, it immediately alerts, drops packets, or tears down the session. In some scenarios it can impersonate the attacker's client to control the Trojan server (e.g. issuing a fake "exit" command), effectively neutralizing the malware remotely
   . In live deployments, this meant that once RAT Catcher caught a Back Orifice or NetBus, those connections were terminated before any real data was exfiltrated.
  </p>
  <p>
   The evaluation showed RAT Catcher works very well
   <i>
    today
   </i>
   , but the authors note challenges ahead. One is strong encryption: some modern RATs use TLS or other protocols, preventing payload inspection. The paper suggests future work in
   <b>
    behavioral and machine-learning methods
   </b>
   , or analyzing pre-encryption handshakes, to cover such cases
   . Another is integration: combining RAT Catcher with firewalls, AVs, or anomaly systems can help fight blended threats (e.g. a worm dropping a RAT) .
  </p>
  <p>
   In summary,
   <b>
    RAT Catcher
   </b>
   demonstrated a powerful concept: by elevating detection to the application layer and stitching together session context, it effectively "catches" RATs that evade conventional defenses. Its 2008 results showed
   <b>
    100% accuracy
   </b>
   on dozens of RAT variants with no false positives. This session-centric approach remains highly relevant for malware analysts today, as adversaries continue to evolve their remote-access tools.
  </p>
  <p>
   
 


