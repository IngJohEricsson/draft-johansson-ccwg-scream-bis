---
docname: draft-johansson-ccwg-rfc8298bis-screamv2-latest
title: Self-Clocked Rate Adaptation for Multimedia
abbrev: SCReAMv2
obsoletes: 8298
cat: exp
ipr: trust200902
wg: CCWG
area: Transport
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"

venue:
  group: Congestion Control Working Group (ccwg)
  mail: ccwg@ietf.org
  github: gloinul/draft-johansson-ccwg-scream-bis

author:
  -

      ins: I. Johansson
      name: Ingemar Johansson
      org: Ericsson
      email: ingemar.s.johansson@ericsson.com
  -
      ins:  M. Westerlund
      name: Magnus Westerlund
      org: Ericsson
      email: magnus.westerlund@ericsson.com

informative:
   RFC2119:
   RFC7478:
   RFC8298:
   RFC8511:
   RFC8699:
   RFC8869:
   RFC8985:
   RFC8257:
   RFC9332:

   Packet-conservation:
      title: Congestion Avoidance and Control
      author:
         -
           ins: V. Jacobson
           name: Van Jacobson
      refcontent: ACM SIGCOMM Computer Communication Review
      seriesinfo:
         DOI: 10.1145/52325.52356
      date: August 1988

   LEDBAT-delay-impact:
      author:
        -
          ins: D. Ros
          name: David Ros
        -
          ins: M. Welzl
          name: Michael Welzl
      refcontent: IEEE Communications Letters, Vol. 17, No. 5,
      seriesinfo:
         DOI: 10.1109/LCOMM.2013.040213.130137
      date: May 2013
      title: Assessing LEDBAT's Delay Impact
      target: http://home.ifi.uio.no/michawe/research/publications/ledbat-impact-letters.pdf

   QoS-3GPP:
      title: "Policy and charging control architecture"
      refcontent: 3GPP TS 23.203
      target: http://www.3gpp.org/ftp/specs/archive/23_series/23.203/
      date: July 2017

   SCReAM-CPP-implementation:
      title: SCReAM - Mobile optimised congestion control algorithm
      author:
         -
            ins: Ericsson Research
      target: https://github.com/EricssonResearch/scream

   SCReAM-evaluation-L4S:
      title: SCReAM - evaluations with L4S
      author:
         -
            ins: Ericsson Research
      target: https://github.com/EricssonResearch/scream/blob/master/L4S-Results.pdf?raw=true

   TFWC:
      title: "Fairer TCP-Friendly Congestion Control Protocol for Multimedia Streaming Applications"
      author:
         -
            ins: S. Choi
            name: Soo-Hyun Choi
         -
            ins: M. Handley
            name: Mark Handley
      seriesinfo:
         DOI: 10.1145/1364654.1364717
      target: http://www-dept.cs.ucl.ac.uk/staff/M.Handley/papers/tfwc-conext.pdf
      date: December 2007

normative:
   RFC3168:
   RFC3550:
   RFC4585:
   RFC5506:
   RFC6298:
   RFC6817:
   RFC8174:
   RFC8888:
   RFC9330:

--- abstract

This memo describes a congestion control algorithm for conversational media
services such as interactive video. The solution conforms to the packet
conservation principle and is a hybrid loss- and delay based congestion control
that also supports ECN and L4S. The algorithm is evaluated over both simulated
Internet bottleneck scenarios as well as in a mobile system simulator using
LongTerm Evolution (LTE) and 5G and is shown to achieve both low latency and
high video throughput in these scenarios. This specification obsoletes RFC
8298. The algorithm supports handling of multiple media streams, typical use
cases are streaming for remote control, AR and 3D VR googles.

--- middle

# Introduction {#introduction}

Congestion in the Internet occurs when the transmitted bitrate is higher than
the available capacity over a given transmission path. Applications that are
deployed in the Internet have to employ congestion control to achieve robust
performance and to avoid congestion collapse in the Internet.

Interactive real-time communication imposes a lot of requirements on the
transport; therefore, a robust, efficient rate adaptation for all access types
is an important part of interactive real-time communications, as the
transmission channel bandwidth can vary over time.

Wireless access such as LTE and 5G, which is an integral part of the current
Internet, increases the importance of rate adaptation as the channel bandwidth
of a default LTE bearer {{QoS-3GPP}} can change considerably in a very short
time frame. Thus, a rate adaptation solution for interactive real-time media,
such as WebRTC {{RFC7478}}, should be both quick and be able to operate over a
large range in channel capacity.

This memo describes Self-Clocked Rate Adaptation for Multimedia version 2
(SCReAMv2), an update to SCReAM congestion control for RTP streams
{{RFC3550}}. While SCReAM was originally devised for WebRTC, SCReAM as well as
SCReAMv2 can also be used for other applications where congestion control of
different type of real-time streams, especially media streams is
necessary. SCReAM is based on the self-clocking principle of TCP and uses
techniques similar to what is used in the rate adaptation algorithm based on Low
Extra Delay Background Transport (LEDBAT) {{RFC6817}}.

SCReAMv2 is not entirely self-clocked as it augments self-clocking with pacing
and a minimum send rate. Further, SCReAMv2 can take advantage of Explicit
Congestion Notification (ECN) {{RFC3168}} and Low Latency Low Loss and Scalable
throughput (L4S) {{RFC9330}} in cases where ECN or L4S is supported by the
network and the hosts. However, ECN or L4S is not required for the basic
congestion control functionality in SCReAMv2.

This specification replaces the previous experimental version {{RFC8298}} of
SCReAM with SCReAMv2. There are many and fairly significant changes to the
original SCReAM algorithm.

The algorithm in this memo differs greatly against the previous version of
SCReAM. The main differences are:

* L4S support added. The L4S algoritm has many similarities with the DCTCP and
  Prague congestion control but has a few extra modifications to make it work
  well with peridic sources such as video.

* The delay based congestion control is changed to implement a pseudo-L4S
  approach, this simplifies the delay based congestion control.

* The fast increase mode is removed. The reference window additive increase is
  replaced with an adaptive multiplicative increase to enhance convergence
  speed.

* The algorithm is more rate based than self-clocked. The calculated congestion
  window is used mainly to calculate proper media bitrates. Bytes in flight is
  however allowed to exceeed the reference window.

* The media bitrate calculation is dramatically changed and simplified.

* Additional compensation is added to make SCReAMv2 handle cases such as large
  changing frame sizes.


## Wireless (LTE and 5G) Access Properties {#lte-5g}

{{RFC8869}} describes the complications that can be observed in wireless
environments. Wireless access such as LTE and 5G typically cannot guarantee a
given bandwidth; this is true especially for default bearers. The network
throughput can vary considerably, for instance, in cases where the wireless
terminal is moving around. Even though 5G can support bitrates above 1 Gbps,
there are cases when the available bitrate can be much lower (less than 10
Mbps); examples are situations with high network load and poor coverage. An
additional complication is that the network throughput can drop for short time
intervals (e.g., at handover); these short glitches are initially very difficult
to distinguish from more permanent reductions in throughput.

Unlike wireline bottlenecks with large statistical multiplexing, it is typically
not possible to try to maintain a given bitrate when congestion is detected with
the hope that other flows will yield. This is because there are generally few
other flows competing for the same bottleneck. Each user gets its own variable
throughput bottleneck, where the throughput depends on factors like channel
quality, network load, and historical throughput. The bottom line is, if the
throughput drops, the sender has no other option than to reduce the
bitrate. Once the radio scheduler has reduced the resource allocation for a
bearer, a flow in that bearer aims to reduce the sending rate quite quickly
(within one RTT) in order to avoid excessive queuing delay or packet loss. This
has the consenquence that L4S capable 5G radio bearers will build a queue unless
these are prioritized over other bearers. This differs from e.g DualQ
{{RFC9332}}, which prioritizes L4S traffic in a weighted scheduler and achives
fairness with additional marking for the L4S flows.

## Why is it a self-clocked algorithm? {#why-selfclock}

Self-clocked congestion control algorithms provide a benefit over their
rate-based counterparts in that the former consists of two adaptation
mechanisms:

* A reference window computation that evolves over a longer timescale (several
  RTTs) especially when the reference window evolution is dictated by estimated
  delay (to minimize vulnerability to, e.g., short-term delay variations). The
  term reference window is used instead of congestion window, as the reference
  window does not set an absolute limit on the bytes in flight.

* A fine-grained congestion control given by the self-clocking; it operates on a
  shorter time scale (1 RTT). The benefits of self-clocking are also elaborated
  upon in {{TFWC}}. The self-clocking however acts more like an emergency break
  as bytes in flight can exceed the reference window only to a certain
  degree. The rationale is to be able to transmit large video frames and avoid
  that they are unnecessarily queued up on the sender side, but still prevent a
  large network queue.

A rate-based congestion control algorithm typically adjusts the rate based on
delay and loss. The congestion detection needs to be done with a certain time
lag to avoid overreaction to spurious congestion events such as delay
spikes. Despite the fact that there are two or more congestion indications, the
outcome is that there is still only one mechanism to adjust the sending
rate. This makes it difficult to reach the goals of high throughput and prompt
reaction to congestion.

## Comparison with LEDBAT and TFWC in TCP {#ledbat-tfwc}

The core SCReAM algorithm, which is still similar in SCReAMv2, has similarities
to the concepts of self-clocking used in TCP-friendly window-based congestion
control {{TFWC}} and follows the packet conservation principle. The packet
conservation principle is described as a key factor behind the protection of
networks from congestion {{Packet-conservation}}.

The reference window is determined in a way similar to the congestion window in
LEDBAT {{RFC6817}}. LEDBAT is a congestion control algorithm that uses send and
receive timestamps to estimate the queuing delay (from now on denoted "qdelay")
along the transmission path. This information is used to adjust the congestion
window. The general problem described in the paper is that the base delay is
offset by LEDBAT's own queue buildup. The big difference with using LEDBAT in
the SCReAM context lies in the facts that the source is rate limited and that
the RTP queue must be kept short (preferably empty). In addition, the output
from a video encoder is rarely constant bitrate; static content (talking heads,
for instance) gives almost zero video bitrate. This yields two useful properties
when LEDBAT's delay-based rate estimation techniques are used as part of SCReAM;
they help to avoid the issues described in {{LEDBAT-delay-impact}}:

1. There is always a certain probability that SCReAM is short of data to
transmit; this means that the network queue will become empty every once in a
while.

2. The max video bitrate can be lower than the link capacity. If the max video
bitrate is 5 Mbps and the capacity is 10 Mbps, then the network queue will
become empty.

It is sufficient that any of the two conditions above is fulfilled to make the
base delay update properly. Furthermore, {{LEDBAT-delay-impact}} describes an
issue with short-lived competing flows. In SCReAM, these short-lived flows will
cause the self-clocking to slow down, thereby building up the RTP queue; in
turn, this results in a reduced media video bitrate. Thus, SCReAM slows the
bitrate more when there are competing short-lived flows than the traditional use
of LEDBAT does. The basic functionality in the use of LEDBAT in SCReAM is quite
simple; however, there are a few steps in order to make the concept work with
conversational media:

* Addition of a media rate control function.

* Reference window validation techniques. The reference window is used as a
  basis for the target bitrate calculation. For that reason, various actions are
  taken to avoid that the reference window grows too much beyond the bytes in
  flight. Additional contraints are applied when in congested state and when the
  max target bitrate is reached.

* Use of inflection points in the reference window calculation to achieve
  reduced delay jitter.

* Adjustment of qdelay target for better performance when competing with other
  loss-based congestion-controlled flows.

The above-mentioned features will be described in more detail in Section
{{scream-detailed-description}}.

The SCReAM/SCReAMv2 congestion control method uses techniques similar to LEDBAT
{{RFC6817}} to measure the qdelay. As is the case with LEDBAT, it is not
necessary to use synchronized clocks in the sender and receiver in order to
compute the qdelay. However, it is necessary that they use the same clock
frequency, or that the clock frequency at the receiver can be inferred reliably
by the sender. Failure to meet this requirement leads to malfunction in the
SCReAM/SCReAMv2 congestion control algorithm due to incorrect estimation of the
network queue delay. Use of {{RFC8888}} as feedback ensures that the same time
base is used in sender and receiver.

# Requirements Language {#requirements}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Overview of SCReAMv2 Algorithm {#scream-overview}


SCReAMv2 still consists of three main parts: network congestion control, sender
transmission control, and media rate control. All of these parts reside at the
sender side. Figure 1 shows the functional overview of a SCReAMv2 sender. The
receiver-side algorithm is very simple in comparison, as it only generates
feedback containing acknowledgements of received RTP packets and indication of
ECN bits.

## Network Congestion Control {#network-cc}

The network congestion control sets an upper limit on how much data can be in
the network (bytes in flight); this limit is called reference window (ref_wnd)
and is used in the sender transmission control.

The sender calculates the reference window based on the feedback from the RTP
receiver. The feedback is timestamp and ECN echo for individual RTP packets.

The receiver of the media echoes a list of received RTP packets and the
timestamp of the RTP packet with the highest sequence number back to the sender
in feedback packets. The sender keeps a list of transmitted packets, their
respective sizes, and the time they were transmitted. This information is used
to determine the number of bytes that can be transmitted at any given time
instant. A reference window puts an upper limit on how many bytes can be in
flight, i.e., transmitted but not yet acknowledged. The reference window is
however not an absolute limit as a slack is given to efficiently transmit
temporary larger media objects, such as video frames.

The reference window seeks to increase by one segment per RTT and this increase
regardless congestion occurs or not, the reference window increase is restriced
or relaxed based on the current value of the reference window relative to a
previous max value and the time elapsed since last congestion event. The
reference window update is increased by one MSS (maximum known RTP packet size)
per RTT with some variation based on reference window size and time elapsed
since the last congestion event. Multiplicative increase allows the congestion
to increase by a fraction of ref_wnd when congestion has not occured for a
while. The reference window is thus an adaptive multiplicative increase that is
mainly additive increase when steady state is reached but allows a faster
convergence to a higher link speed.

Reference window reduction is triggered by:

* Packet loss or Classic ECN marking is detected : The reference window is
  reduced by a predetermined fraction.

* Estimated queue delay exceeds a given threshold : The reference window is
  reduced given by how much the delay exceeds the threshold.

* L4S ECN marking detected : The reference window is reduced in proportion to
  the fraction of packets that are marked (scalable congestion control).

## Sender Transmission Control {#sender-tc}

The sender transmission control limits the output of data, given by the relation
between the number of bytes in flight and the reference window. The congestion
window is however not a hard limit, additional slack is given to avoid that RTP
packets are queued up unnecessarily on the sender side. This means that the
algoritm prefers to build up a queue in the network rather than on the sender
side. Additional congestion that this causes will reflect back and cause a
reduction of the reference window. Packet pacing is used to mitigate issues with
ACK compression that MAY cause increased jitter and/or packet loss in the media
traffic. Packet pacing limits the packet transmission rate given by the
estimated link throughput. Even if the send window allows for the transmission
of a number of packets, these packets are not transmitted immediately; rather,
they are transmitted in intervals given by the packet size and the estimated
link throughput. Packets are generally paced at a higher rate than the target
bitrate, this makes it possible to transmit occasionally larger video frames in
a timely manner.

## Media Rate Control {#media-rate-control}

The media rate control serves to adjust the media bitrate to ramp up quickly
enough to get a fair share of the system resources when link throughput
increases.

The reaction to reduced throughput must be prompt in order to avoid getting too
much data queued in the RTP packet queue(s) in the sender. The media rate is
calculated based on the reference window and RTT. For the case that multiple
streams are enabled, the media rate among the streams is distrubuted according
to relative priorities.

In cases where the sender's frame queues increase rapidly, such as in the case
of a Radio Access Type (RAT) handover, the SCReAMv2 sender MAY implement
additional actions, such as discarding of encoded media frames or frame skipping
in order to ensure that the RTP queues are drained quickly. Frame skipping
results in the frame rate being temporarily reduced. Which method to use is a
design choice and is outside the scope of this algorithm description.

# Detailed Description of SCReAMv2 sender algorithm {#scream-detailed-description}

This section describes the sender-side algorithm in more detail. It is split
between the network congestion control, sender transmission control, and media
rate control.

The sender implements media rate control and an RTP queue for each media type or
source, where RTP packets containing encoded media frames are temporarily stored
for transmission. Figure 1 shows the details when a single media source (or
stream) is used. A transmission scheduler (not shown in the figure) is added to
support multiple streams. The transmission scheduler can enforce differing
priorities between the streams and act like a coupled congestion controller for
multiple flows. Support for multiple streams is implemented in
{{SCReAM-CPP-implementation}}.

~~~aasvg
          +---------------------------+
          |        Media encoder      |
          +---------------------------+
              ^                  |(1)
              |                 RTP
              |(3)               |
              |                  V
              |            +-----------+
         +---------+       |           |
         | Media   |       |   Queue   |
         | rate    |       |           |
         | control |       |RTP packets|
         +---------+       |           |
              ^            +-----------+
              |                  |
              | (2)              |(4)
              |                 RTP
              |                  |
              |                  v
    +------------+       +--------------+
    |  Network   |  (7)  |    Sender    |
+-->| congestion |------>| Transmission |
|   |  control   |       |   Control    |
|   +------------+       +--------------+
|                                |(5)
+-------------RTCP----------+   RTP
    (6)                     |    |
                            |    v
                        +------------+
                        |     UDP    |
                        |   socket   |
                        +------------+
~~~
{: #fig-sender-view title="Sender Functional View"}

Media frames are encoded and forwarded to the RTP queue (1) in
{{fig-sender-view}}. The RTP packets are picked from the RTP queue (4), for
multiple flows from each RTP queue based on some defined priority order or
simply in a round-robin fashion, by the sender transmission controller.

The sender transmission controller (in case of multiple flows a transmission
scheduler) sends the RTP packets to the UDP socket (5). In the general case, all
media SHOULD go through the sender transmission controller and is limited so
that the number of bytes in flight is less than the reference window albeit with
a slack to avoid that packets are unnecessarily delayed in the RTP queue.

RTCP packets are received (6) and the information about the bytes in flight and
reference window is exchanged between the network congestion control and the
sender transmission control (7).

The reference window and the estimated RTT is communicated to the media rate
control (2) to compute the appropriate target bitrate. The target bitrate is
updated whenever the reference window is updated. Additional parameters are also
communicated to make the rate control more stable when the congestion window is
very small or when L4S is not active. This is described more in detail below.

## Constants and variables {#constants-variables}

Constants and state variables are listed in this section. Temporary variables
are not listed; instead, they are appended with '_t' in the pseudocode to
indicate their local scope.

### Constants {#constants}

The RECOMMENDED values, within parentheses "()", for the constants are deduced
from experiments.

* QDELAY_TARGET_LO (0.06): Target value for the minimum qdelay [s].

* QDELAY_TARGET_HI (0.4): Target value for the maximum qdelay [s]. This
  parameter provides an upper limit to how much the target qdelay
  (qdelay_target) can be increased in order to cope with competing loss-based
  flows. However, the target qdelay does not have to be initialized to this high
  value, as it would increase end-to-end delay and also make the rate control
  and congestion control loops sluggish.

* MIN_REF_WND (3000): Minimum reference window [byte].

* MAX_BYTES_IN_FLIGHT_HEAD_ROOM (1.1): Headroom for the limitation of ref_wnd.

* BETA_LOSS (0.7): ref_wnd scale factor due to loss event.

* BETA_ECN (0.8): ref_wnd scale factor due to ECN event.

* MSS (1000 byte): Maximum segment size = Max RTP packet size.

* TARGET_BITRATE_MIN: Minimum target bitrate in [bps] (bits per second).

* TARGET_BITRATE_MAX: Maximum target bitrate in [bps].

* RATE_PACE_MIN (50000): Minimum pacing rate in [bps].

* REF_WND_OVERHEAD (1.5): Indicates how much bytes in flight is allowed to
  exceed ref_wnd.

* L4S_AVG_G (1.0/16): Exponentially
  Weighted Moving Average (EWMA) factor for l4s_alpha

* QDELAY_AVG_G (1.0/4): Exponentially
  Weighted Moving Average (EWMA) factor for qdelay_avg

* POST_CONGESTION_DELAY (4.0): Determines how long (seconds) after a congestion
  event that the reference window growth should be cautious.

* MUL_INCREASE_FACTOR (0.02): Determines how much (as a fraction of ref_wnd)
  that the ref_wnd can increase per RTT.

* IS_L4S (false): Congestion control operates in L4S mode.

* VIRTUAL_RTT (0.025): Virtual RTT [s]

* PACKET_PACING_HEADROOM (1.5): Extra head room for packet pacing.

* BYTES_IN_FLIGHT_HEAD_ROOM (2.0): Extra headroom for bytes in flight.

### State Variables {#state-variables}

The values within parentheses "()" indicate initial values.

* qdelay_target (QDELAY_TARGET_LO): qdelay target [s], a variable qdelay target
  is introduced to manage cases where a fixed qdelay target would otherwise
  starve the data flow under such circumstances (e.g., FTP competes for the
  bandwidth over the same bottleneck). The qdelay target is allowed to vary
  between QDELAY_TARGET_LO and QDELAY_TARGET_HI.

* qdelay_fraction_avg (0.0): Fractional qdelay filtered by the Exponentially
  Weighted Moving Average (EWMA).

* ref_wnd (MIN_REF_WND): Reference window.

* ref_wnd_i (1): Reference window inflection point.

* bytes_newly_acked (0): The number of bytes that was acknowledged with the last
  received acknowledgement, i.e., bytes acknowledged since the last ref_wnd
  update.

* max_bytes_in_flight (0): The maximum number of bytes in flight in the last
  round trip.

* max_bytes_in_flight_prev (0): The maximum number of bytes in flight in
  previous round trip.

* send_wnd (0): Upper limit to how many bytes can currently be
  transmitted. Updated when ref_wnd is updated and when RTP packet is
  transmitted.

* target_bitrate (0): Media target bitrate [bps].

* rate_media (0.0): Measured bitrate [bps] from the media encoder.

* s_rtt (0.0): Smoothed RTT [s], computed with a similar method to that
  described in {{RFC6298}}.

* rtp_size (0): Size [byte] of the last transmitted RTP packet.

* loss_event_rate (0.0): The estimated fraction of RTTs with lost packets
  detected.

* bytes_in_flight_ratio (0.0): Ratio between the bytes in flight and the
  reference window.

* ref_wnd_ratio (0.0): Ratio between MSS and ref_wnd.

* last_fraction_marked (0.0): fraction marked packets in last update

* l4s_alpha (0.0): Average fraction of marked packets per RTT.

* l4s_active (false): Indicates that L4S is enabled and packets are indeed
  marked.

* last_update_l4s_alpha_time (0): Last time l4s_alpha was updated [s].

* last_update_qdelay_avg_time (0): Last time qdelay_avg was updated [s].

* packets_delivered_this_rtt (0): Counter for delivered packets.

* packets_marked_this_rtt (0): Counter delivered and ECN-CE marked packets.

* last_congestion_detected_time (0): Last time congestion event occured [s].

* last_ref_wnd_i_update_time (0): Last time ref_wnd_i was updated [s].

* bytes_newly_acked (0): Number of bytes newly ACKed, reset to 0 when congestion
  window is updated [byte].

* bytes_newly_acked_ce (0): Number of bytes newly ACKed and CE marked, reset to
  0 when reference window is updated [byte].

* pace_bitrate (1e6): Packet pacing rate [bps].

* t_pace (1e-6): Pacing interval between packets [s].

* rel_framesize_high (1.0): High percentile of frame size, normalized by nominal
  frame size for the given target bitrate

* frame_period (0.02): The frame period [s].


## Network Congestion Control {#network-cc-2}

This section explains the network congestion control, which performs two main
functions:

* Computation of reference window at the sender: This gives an upper limit to
  the number of bytes in flight.

* Calculation of send window at the sender: RTP packets are transmitted if
  allowed by the relation between the number of bytes in flight and the
  reference window. This is controlled by the send window.

SCReAMv2 is a window-based and byte-oriented congestion control protocol, where
the number of bytes transmitted is inferred from the size of the transmitted RTP
packets. Thus, a list of transmitted RTP packets and their respective
transmission times (wall-clock time) MUST be kept for further calculation.

The number of bytes in flight (bytes_in_flight) is computed as the sum of the
sizes of the RTP packets ranging from the RTP packet most recently transmitted,
down to but not including the acknowledged packet with the highest sequence
number. This can be translated to the difference between the highest transmitted
byte sequence number and the highest acknowledged byte sequence number. As an
example: If an RTP packet with sequence number SN is transmitted and the last
acknowledgement indicates SN-5 as the highest received sequence number, then
bytes_in_flight is computed as the sum of the size of RTP packets with sequence
number SN-4, SN-3, SN-2, SN-1, and SN. It does not matter if, for instance, the
packet with sequence number SN-3 was lost -- the size of RTP packet with
sequence number SN-3 will still be considered in the computation of
bytes_in_flight.

bytes_in_flight_ratio is calculated as the ratio between bytes_flight and
ref_wnd. This value should be computed at the beginning of the ACK
processing. ref_wnd_ratio is computed as the relation between MSS and ref_wnd.

Furthermore, a variable bytes_newly_acked is incremented with a value
corresponding to how much the highest sequence number has increased since the
last feedback. As an example: If the previous acknowledgement indicated the
highest sequence number N and the new acknowledgement indicated N+3, then
bytes_newly_acked is incremented by a value equal to the sum of the sizes of RTP
packets with sequence number N+1, N+2, and N+3. Packets that are lost are also
included, which means that even though, e.g., packet N+2 was lost, its size is
still included in the update of bytes_newly_acked. The bytes_newly_acked_ce is,
similar to bytes_newly_acked, a counter of bytes newly acked with the extra
condition that they are ECN-CE marked. The bytes_newly_acked and
bytes_newly_acked_ce are reset to zero after a ref_wnd update.

The feedback from the receiver is assumed to consist of the following elements.

* A list of received RTP packets' sequence numbers. With an indication if
  packets are ECN-CE marked.

* The wall-clock timestamp corresponding to the received RTP packet with the
  highest sequence number.

It is recommended to use RFC8888 {{RFC8888}} for the feedback as it supports the
feedback elements described above.

When the sender receives RTCP feedback, the qdelay is calculated as outlined in
{{RFC6817}}. A qdelay sample is obtained for each received acknowledgement. A
number of variables are updated as illustrated by the pseudocode below;
temporary variables are appended with '_t'. Division operation is always
floating point unless otherwise noted. l4s_alpha is calculated based in number
of packets delivered (and marked). This makes calculation of L4S alpha more
accurate at very low bitrates, given that the tail RTP packet in a video frame
is often smaller than MSS.

The smoothed RTT (s_rtt) is computed in a way similar to {{RFC6298}}.

~~~
packets_delivered_this_rtt += packets_acked
packets_marked_this_rtt += packets_acked_ce
# l4s_alpha is updated at least every 10ms
if (now - last_update_l4s_alpha_time >= min(0.01,s_rtt)
  # l4s_alpha is calculated from packets marked istf bytes marked
  fraction_marked_t = packets_marked_this_rtt/
                      packets_delivered_this_rtt

  l4s_alpha = L4S_AVG_G*fraction_marked_t + (1.0-L4S_AVG_G)*l4S_alpha

  last_update_l4s_alpha_time = now
  packets_delivered_this_rtt = 0
  packets_marked_this_rtt = 0
  last_fraction_marked = fraction_marked_t
end

if (now - last_update_qdelay_avg_time >= s_rtt)
  # qdelay_avg is updated with a slow attack, fast decay EWMA filter
  if (qdelay < qdelay_avg)
    qdelay_avg = qdelay
  else
    qdelay_avg = QDELAY_AVG_G*qdelay + (1.0-QDELAY_AVG_G)*qdelay_avg
  end
  last_update_qdelay_avg_time = now
end
~~~

### Reaction to Delay, Packet Loss and ECN-CE {#reaction-delay-loss-ce}

Congestion is detected based on three different indicators:

 * Lost packets detected. The loss detection is described in {{detect-loss}}.

 * ECN-CE marked packets detected.

 * Estimated queue delay exceeds a threshold.

A congestion event occurs if any of the above indicators are true AND it is at
least min(VIRTUAL_RTT,s_rtt) since the last congestion event. This ensures that
the reference window is reduced at most once per smoothed RTT.

#### Lost packets  {#reaction-loss}

The reference window back-off due to loss events is deliberately a bit less than
is the case with TCP Reno, for example. TCP is generally used to transmit whole
files; the file is then like a source with an infinite bitrate until the whole
file has been transmitted. SCReAMv2, on the other hand, has a source which rate
is limited to a value close to the available transmit rate and often below that
value; the effect is that SCReAMv2 has less opportunity to grab free capacity
than a TCP-based file transfer. To compensate for this, it is RECOMMENDED to let
SCReAMv2 reduce the reference window less than what is the case with TCP when
loss events occur.

#### ECN-CE and classic ECN  {#reaction-ecn-ce}

In classic ECN mode the ref_wnd is scaled by a fixed value (BETA_ECN).

The reference window back-off due to an ECN event MAY be smaller than if a loss
event occurs. This is in line with the idea outlined in {{RFC8511}} to enable
ECN marking thresholds lower than the corresponding packet drop thresholds.

#### ECN-CE and L4S {#reaction-l4s-ce}

The ref_wnd is scaled down in proportion to the fraction of marked packets per
RTT. The scale down proportion is given by l4s_alpha, which is an EWMA filtered
version of the fraction of marked packets per RTT. This is inline with how DCTCP
works {{RFC8257}}. Additional methods are applied to make the reference window
reduction reasonably stable, especially when the reference window is only a few
MSS. In addition, because SCReAMv2 can quite often be source limited, additional
steps are taken to restore the reference window to a proper value after a long
period without congestion.

#### Increased queue delay {#reaction-delay}

SCReAMv2 implements a delay based congestion control approach where it mimics
L4S congestion marking when the averaged queue delay exceeds a target
threshold. This threshold is set to qdelay_target/2 and the congestion backoff
factor (l4s_alpha_v) increases linearly from 0 to 100% as qdelay_avg goes from
qdelay_target/2 to qdelay_target. The averaged qdelay (qdelay_avg) is used to
avoid that the SCReAMv2 congestion control over-reacts to scheduling jitter,
sudden delay spikes due to e.g. handover or link layer
retransmissions. Furthermore, the delay based congestion control is inactivated
when it is reasonably certain that L4S is active, i.e. L4S is enabled and
congested nodes apply L4S marking of packets. This reduces negative effects of
clockdrift, that the delay based control can introduce, whenever possible.

### Reference Window Update {#ref-wnd-update}

The reference window update contains two parts. One that reduces the congestion
window when congestion events (listed above) occur, and one part that
continously increases the reference window.

The target bitrate is updated whenever the reference window is updated.

Actions when congestion detected

~~~
# The reference window is updated at least every VIRTUAL_RTT
if (now - last_congestion_detected_time >= min(VIRTUAL_RTT,s_rtt)
  if (loss detected)
    is_loss_t = true
  end
  if (packets marked)
    is_ce_t = true
  end
  if (qdelay > qdelay_target/2)
    # It is expected that l4s_alpha is below a given value,
    l4_alpha_lim_t = 2 / target_bitrate * MSS * 8 / s_rtt
    if (l4s_alpha < l4_alpha_lim_t || !l4s_active)
      # L4S does not seem to be active
      l4s_alpha_v_t = min(1.0, max(0.0,
         (qdelay_avg - qdelay_target / 2) /
         (qdelay_target / 2)));
      is_virtual_ce_t = true
    end
  end
end

if (is_loss_t || is_ce_t || is_virtual_ce_t)
  if (now - last_ref_wnd_i_update_time > 0.25)
    # Update ref_wnd_i
    # Additional median filtering over more congestion epochs
    # may improve accuracy of ref_wnd_i
    last_ref_wnd_i_update_time = now
    ref_wnd_i = ref_wnd
  end
end


# Either loss, ECN mark or increased qdelay is detected
if (is_loss_t)
  # Loss is detected
  ref_wnd = ref_wnd * BETA_LOSS
end
if (is_ce_t)
  # ECN-CE detected
  if (IS_L4S)
    # L4S mode
    backoff_t = l4s_alpha / 2

    # Increase stability for very small ref_wnd
    backOff_t *= max(0.8, 1.0 - ref_wnd_ratio * 2)

    if (now - last_congestion_detected_time > 5)
      # A long time since last congested because link throughput
      # exceeds max video bitrate.
      # There is a certain risk that ref_wnd has increased way above
      # bytes in flight, so we reduce it here to get it better on
      # track and thus the congestion episode is shortened
      ref_wnd = min(ref_wnd, max_bytes_in_flight_prev)

      # Also, we back off a little extra if needed
      # because alpha is quite likely very low
      # This can in some cases be an over-reaction
      # but as this function should kick in relatively seldom
      # it should not be to too big concern
      backoff_t = max(backoff_t, 0.25)

      # In addition, bump up l4sAlpha to a more credible value
      # This may over react but it is better than
      # excessive queue delay
      l4sAlpha = 0.25
    end
    ref_wnd = (1.0 - backoff_t) * ref_wnd
  else
    # Classic ECN mode
    ref_wnd = ref_wnd * BETA_ECN
  end
end
if (is_virtual_ce_t)
  backoff_t = l4s_alpha_v_t / 2
  ref_wnd = (1.0 - backoff_t) * ref_wnd
end
ref_wnd = max(MIN_REF_WND, ref_wnd)

if (is_loss_t || is_ce_t || is_virtual_ce_t)
  last_congestion_detected_time = now
end
~~~

The variable max_bytes_in_flight_prev indicates the maximum bytes in flights in
the previous round trip. The reason to this is that bytes in flight can spike
when congestion occurs, max_bytes_in_flight_prev thus ensures better that an
uncongested bytes in flight is used.


Reference window increase

~~~
# Delay factor for multiplicative reference window increase
# after congestion

post_congestion_scale_t = max(0.0, min(1.0, (now -
last_congestion_detected_time) / POST_CONGESTION_DELAY))

# Scale factor for ref_wnd update
ref_wnd_scale_factor_t = 1.0 + (MUL_INCREASE_FACTOR  * ref_wnd) / MSS)


# Calculate bytes acked that are not CE marked
bytes_newly_acked_minus_ce_t = bytes_newly_acked-
                               bytes_newly_acked_ce

increment_t = bytes_newly_acked_minus_ce_t*ref_wnd_ratio

# Reduce increment for small RTTs
tmp_t = min(1.0, s_rtt / VIRTUAL_RTT)
increment_t *= tmp_t * tmp_t

# Apply limit to reference window growth when close to last
# known max value before congestion
scl_t = (ref_wnd-ref_wnd_i) / ref_wnd_i
scl_t *= 4
scl_low_limit_t = 0.1
if (is_l4s_active)
  # Restrict limit to small reference windows. The limitation
  # is gradually lifted and completely removed when
  # ref_wnd > 50MSS
  scl_low_limit_t = max(0.1, min(1.0, 0.02 / ref_wnd_ratio));
end

scl_t = scl_t * scl_t
scl_t = max(scl_low_limit_t, min(1.0, scl_t))
increment_t *= scl_t

# Slow down ref_wnd increase when ref_wnd is only a few MSS
# The ref_wnd increase can be as slow as
# LOW_REF_WND_SCALE_FACTOR*MSS per RTT
float tmp_t = ref_wnd_scale_factor_t

# Further limit multiplicative increase when congestion occured
# recently.
if (tmp_t > 1.0)
  tmp_t = 1.0 + (tmp_t - 1.0) * post_congestion_scale_t * scl_t;
end
increment *= tmp_t

# Increase ref_wnd only if bytes in flight is large enough
# Quite a lot of slack is allowed here to avoid that bitrate
# locks to low values.
max_allowed_t = MSS + max(max_bytes_in_flight,
  max_bytes_in_flight_prev) * BYTES_IN_FLIGHT_HEAD_ROOM
int ref_wnd_t = ref_wnd + increment_t
if (ref_wnd_t <= max_allowed_t)
  ref_wnd = ref_wnd_t
end
~~~

The ref_wnd_scale_factor_t scales the reference window increase. The
ref_wnd_scale_factor_t is increased with larger ref_wnd to allow for a
multiplicative increase and thus a faster convergence when link capacity
increases.

The variable max_bytes_in_flight indicates the max bytes in flight in the
current round trip.

The multiplicative increase is restricted directly after a congestion event and
the restriction is gradually relaxed as the time since last congested
increased. The restriction makes the reference window growth to be no faster
than additive increase when congestion continusly occurs.  For L4S operation
this means that the SCReAMv2 algorithm will adhere to the 2 marked packets per
RTT equilibrium at steady state congestion, with the exception of the case
below.

The reference window increase is restricted to values as small as 0.1MSS/RTT
when the reference window is close to the last known max value (ref_wnd_i). This
increases stability and reduces periodic overshoot. This restriction is applied
in full only for small reference windows when in L4S operation.

It is particularly important that the reference window reflects the transmitted
bitrate especially in L4S mode operation. An inflated ref_wnd takes extra RTTs
to bring down to a correct value upon congestion and thus causes unnecessary
queue buildup. At the same time the reference window must be allowed to be large
enough to avoid that the SCReAMv2 algorithm begins to limit itself, given that
the target bitrate is calculated based on the ref_wnd. Two mechanisms are used
to manage this:

* Restore correct value of ref_wnd upon congestion. This is done if is a
  prolonged time since the link was congested. A typical example is that
  SCReAMv2 has been rate limited, i.e the target bitrate has reached the
  TARGET_BITRATE_MAX.

* Limit ref_wnd when the target_bitrate has reached TARGET_BITRATE_MAX. The
  ref_wnd is restricted based on a history of the last max_bytes_in_flight
  values. See {{SCReAM-CPP-implementation}} for details.

The two mechanisms complement one another.

### Lost Packet Detection {#detect-loss}

Lost packet detection is based on the received sequence number list. A
reordering window SHOULD be applied to prevent packet reordering from triggering
loss events. The reordering window is specified as a time unit, similar to the
ideas behind Recent ACKnowledgement (RACK) {{RFC8985}}. The computation of the
reordering window is made possible by means of a lost flag in the list of
transmitted packets. This flag is set if the received sequence number list
indicates that the given packet is missing. If later feedback indicates that
a previously lost marked packet was indeed received, then the reordering window
is updated to reflect the reordering delay. The reordering window is given by
the difference in time between the event that the packet was marked as lost and
the event that it was indicated as successfully received. Loss is detected if a
given RTP packet is not acknowledged within a time window (indicated by the
reordering window) after an packet with a higher sequence number was
acknowledged.

### Send Window Calculation {#send-window}

The basic design principle behind packet transmission in SCReAM was to allow
transmission only if the number of bytes in flight is less than the congestion
window. There are, however, two reasons why this strict rule will not work
optimally:

* Bitrate variations: Media sources such as video encoders generally produce
  frames whose size always vary to a larger or smaller extent. The RTP queue
  absorbs the natural variations in frame sizes. However, the RTP queue should
  be as short as possible to prevent the end-to-end delay from increasing. A
  strict 'send only when bytes in flight is less than the reference window' rule
  can cause the RTP queue to grow simply because the send window is limited. The
  consequence is that the reference window will not increase, or will increase
  very slowly, because the reference window is only allowed to increase when
  there is a sufficient amount of data in flight. The final effect is that the
  media bitrate increases very slowly or not at all.

* Reverse (feedback) path congestion: Especially in transport over
  buffer-bloated networks, the one-way delay in the reverse direction can jump
  due to congestion. The effect is that the acknowledgements are delayed, and
  the self-clocking is temporarily halted, even though the forward path is not
  congested. The REF_WND_OVERHEAD allows for some degree of reverse path
  congestion as the bytes in flight is allowed to exceed ref_wnd.


In SCReAMv2, the send window is given by the relation between the adjusted
reference window and the amount of bytes in flight according to the pseudocode
below. The multiplication of ref_wnd with REF_WND_OVERHEAD and
rel_framesize_high has the effect that bytes in flight is 'around' the ref_wnd
rather than limited by the ref_wnd when the link is congested.  The
implementation allows the RTP queue to be small even when the frame sizes vary
and thus increased e2e delay can be avoided.

~~~
send_wnd = ref_wnd * REF_WND_OVERHEAD * rel_framesize_high -
           bytes_in_flight
~~~

The send window is updated whenever an packet is transmitted or an feedback
messaged is received.

The variable rel_framesize_high is based on calculation of the high percentile
of the frame sizes. The calculation is based on a histogram of the frame sizes
relative to the expected frame size given the target bitrate and frame
period. The calculation of rel_framesize_high is done for every new video frame
and is outlined roughly with the pseudo code below. For more detailed code, see
{{SCReAM-CPP-implementation}}.

~~~
# frame_size is that frame size for the last encoded frame
tmp_t = frame_size / (target_bitrate * frame_period / 8)

if (tmp_t > 1.0)
  # Insert sample into histogram
  insert_into_histogram(tmp_t)
  # Get high percentile
  rel_framesize_high = get_histogram_high_percentile()
end
~~~

A 75%-ile is used in {{SCReAM-CPP-implementation}}, the histogram can be made
leaky such that old samples are gradually forgotten.


### Packet Pacing {#packet-pacing}

Packet pacing is used in order to mitigate coalescing, i.e., when packets are
transmitted in bursts, with the risks of increased jitter and potentially
increased packet loss. Packet pacing is also recommended to be used with L4S and
also mitigates possible issues with queue overflow due to key-frame generation
in video coders. The time interval between consecutive packet transmissions is
greater than or equal to t_pace, where t_pace is given by the equations below :

~~~
pace_bitrate = max(RATE_PACE_MIN, target_bitrate) *
               PACKET_PACING_HEADROOM
t_pace = rtp_size * 8 / pace_bitrate
~~~

rtp_size is the size of the last transmitted RTP packet. RATE_PACE_MIN is the
minimum pacing rate.

### Stream Prioritization {#stream-prioritization}

The SCReAM algorithm makes a distinction between network congestion control and
media rate control. This is easily extended to many streams. RTP packets from
two or more RTP queues are scheduled at the rate permitted by the network
congestion control.

The scheduling can be done by means of a few different scheduling regimes. For
example, the method for coupled congestion control specified in {{RFC8699}} can
be used. One implementation of SCReAMv2 {{SCReAM-CPP-implementation}} uses
credit-based scheduling. In credit-based scheduling, credit is accumulated by
queues as they wait for service and is spent while the queues are being
serviced. For instance, if one queue is allowed to transmit 1000 bytes, then a
credit of 1000 bytes is allocated to the other unscheduled queues. This
principle can be extended to weighted scheduling, where the credit allocated to
unscheduled queues depends on the relative weights. The latter is also
implemented in {{SCReAM-CPP-implementation}} in which case the target bitrate
for the streams are also scaled relative to the scheduling priority.

## Media Rate Control {#media-rate-control-2}

The media rate control algorithm is executed whenever the reference window is
updated and updates the target bitrate. The target bitrate is essentiatlly based
on the reference window and the (smoothed) RTT according to

~~~
target_bitrate = 8 * ref_wnd / s_rtt
~~~

The role of the media rate control is to strike a reasonable balance between a
low amount of queuing in the RTP queue(s) and a sufficient amount of data to
send in order to keep the data path busy. Because the reference window is
updated based on loss, ECN-CE and delay, so does the target rate also update.

The code above however needs some modifications to work fine in a number of
scenarios

* L4S is inactive, i.e L4S is either not enabled or congested bottlenecks do not
  L4S mark packets

* ref_wnd is very small, just a few MSS or smaller

The complete pseudo code for adjustment of the target bitrate is shown below

~~~
tmp_t = 1.0

# limit bitrate if bytes in flight exceeds is close to or
# exceeds ref_wnd. This helps to avoid large rate fluctiations and
# variations in RTT
# Only applied when L4S is inactive
if (!l4s_active && bytes_in_flight_ratio > BYTES_IN_FLIGHT_LIMIT)
  tmp_t /= min(BYTES_IN_FLIGHT_LIMIT_COMPENSATION,
    bytesInFlightRatio / BYTES_IN_FLIGHT_LIMIT)
end

# Scale down rate slighty when the reference window is very
# small compared to MSS
tmp_t *= 1.0 - min(0.8, max(0.0, ref_wnd_ratio - 0.1))

# Calculate target bitrate and limit to min and max allowed
# values
target_bitrate = tmp_t * 8 * ref_wnd / s_rtt
target_bitrate = min(TARGET_BITRATE_MAX,
  max(TARGET_BITRATE_MIN,target_bitrate))
~~~


## Competing Flows Compensation {#competing-flows-compensation}

It is likely that a flow will have to share congested bottlenecks with other
flows that use a more aggressive congestion control algorithm (for example,
large FTP flows using loss-based congestion control). The worst condition occurs
when the bottleneck queues are of tail-drop type with a large buffer
size. SCReAMv2 takes care of such situations by adjusting the qdelay_target when
loss-based flows are detected, as shown in the pseudocode below.

~~~
adjust_qdelay_target(qdelay)
  qdelay_norm_t = qdelay / QDELAY_TARGET_LOW
  update_qdelay_norm_history(qdelay_norm_t)
  # Compute variance
  qdelay_norm_var_t = VARIANCE(qdelay_norm_history(200))
  # Compensation for competing traffic
  # Compute average
  qdelay_norm_avg_t = AVERAGE(qdelay_norm_history(50))
  # Compute upper limit to target delay
  new_target_t = qdelay_norm_avg_t + sqrt(qdelay_norm_var_t)
  new_target_t *= QDELAY_TARGET_LO
  if (loss_event_rate > 0.002)
    # Packet losses detected
    qdelay_target = 1.5 * new_target_t
  else
    if (qdelay_norm_var_t < 0.2)
      # Reasonably safe to set target qdelay
      qdelay_target = new_target_t
    else
      # Check if target delay can be reduced; this helps prevent
      # the target delay from being locked to high values forever
      if (new_target_t < QDELAY_TARGET_LO)
        # Decrease target delay quickly, as measured queuing
        # delay is lower than target
        qdelay_target = max(qdelay_target * 0.5, new_target_t)
      else
        # Decrease target delay slowly
        qdelay_target *= 0.9
      end
    end
  end

  # Apply limits
  qdelay_target = min(QDELAY_TARGET_HI, qdelay_target)
  qdelay_target = max(QDELAY_TARGET_LO, qdelay_target)
~~~

Two temporary variables are calculated. qdelay_norm_avg_t is the long-term
average queue delay, qdelay_norm_var_t is the long-term variance of the queue
delay. A high qdelay_norm_var_t indicates that the queue delay changes; this can
be an indication that bottleneck bandwidth is reduced or that a competing flow
has just entered. Thus, it indicates that it is not safe to adjust the queue
delay target.

A low qdelay_norm_var_t indicates that the queue delay is relatively stable. The
reason could be that the queue delay is low, but it could also be that a
competing flow is causing the bottleneck to reach the point that packet losses
start to occur, in which case the queue delay will stay relatively high for a
longer time.

The queue delay target is allowed to be increased if either the loss event rate
is above a given threshold or qdelay_norm_var_t is low. Both these conditions
indicate that a competing flow may be present. In all other cases, the queue
delay target is decreased.

The function that adjusts the qdelay_target is simple and could produce false
positives and false negatives. The case that self-inflicted congestion by the
SCReAMv2 algorithm may be falsely interpreted as the presence of competing
loss-based FTP flows is a false positive. The opposite case -- where the
algorithm fails to detect the presence of a competing FTP flow -- is a false
negative.

Extensive simulations have shown that the algorithm performs well in LTE and 5G
test cases and that it also performs well in simple bandwidth-limited bottleneck
test cases with competing FTP flows. However, the potential failure of the
algorithm cannot be completely ruled out. A false positive (i.e., when
self-inflicted congestion is mistakenly identified as competing flows) is
especially problematic when it leads to increasing the target queue delay, which
can cause the end-to-end delay to increase dramatically.

If it is deemed unlikely that competing flows occur over the same bottleneck,
the algorithm described in this section MAY be turned off. One such case is
QoS-enabled bearers in 3GPP-based access such as LTE and 5G. However, when
sending over the Internet, often the network conditions are not known for sure,
so in general it is not possible to make safe assumptions on how a network is
used and whether or not competing flows share the same bottleneck. Therefore,
turning this algorithm off must be considered with caution, as it can lead to
basically zero throughput if competing with loss-based traffic.

## Handling of systematic errors in video coders {#coder-errors}

Some video encoders are prone to systematically generate an output bitrate that
is systematically larger or smaller than the target bitrate. SCReAMv2 can handle
some deviation inherently but for larger devation it becomes necessary to
compensate for this. The algorithm for this is detailed in
{{SCReAM-CPP-implementation}}.

ToDo: A future draft version will describe this in more detail as it has been
fully integrated into SCReAMv2.

#  Receiver Requirements on Feedback Intensity {#scream-receiver}

The simple task of the receiver is to feed back acknowledgements with with time
stamp and ECN bits indication for received packets to the sender. Upon reception
of each RTP packet, the receiver MUST maintain enough information to send the
aforementioned values to the sender via an RTCP transport- layer feedback
message. The frequency of the feedback message depends on the available RTCP
bandwidth. The requirements on the feedback elements and the feedback interval
are described below.

SCReAMv2 benefits from relatively frequent feedback. It is RECOMMENDED that a
SCReAMv2 implementation follows the guidelines below.

The feedback interval depends on the media bitrate. At low bitrates, it is
sufficient with a feedback every frame; while at high bitrates, a feedback
interval of roughly 5ms is preferred. At very high bitrates, even shorter
feedback intervals MAY be needed in order to keep the self-clocking in SCReAMv2
working well. One indication that feedback is too sparse is that the SCReAMv2
implementation cannot reach high bitrates, even in uncongested links. More
frequent feedback might solve this issue.

The numbers above can be formulated as a feedback interval function that can be
useful for the computation of the desired RTCP bandwidth. The following equation
expresses the feedback rate:

~~~
# Assume 100 byte RTCP packets
rate_fb = 0.02 * [average received rate] / (100.0 * 8.0);
rate_fb = min(1000, max(10, rate_fb))

# Calculate feedback intervals
fb_int = 1.0/rate_fb
~~~

Feedback should also forcibly be transmitted in any of these cases:

* More than N packets received since last RTCP feedback has been transmitted

* An RTP packet with marker bit set is received

The transmission interval is not critical. So, in the case of multi-stream
handling between two hosts, the feedback for two or more synchronization sources
(SSRCs) can be bundled to save UDP/IP overhead. However, the final realized
feedback interval SHOULD not exceed 2*fb_int in such cases, meaning that a
scheduled feedback transmission event should not be delayed more than fb_int.

SCReAMv2 works with AVPF regular mode; immediate or early mode is not required
by SCReAMv2 but can nonetheless be useful for RTCP messages not directly related
to SCReAMv2, such as those specified in {{RFC4585}}. It is RECOMMENDED to use
reduced-size RTCP {{RFC5506}}, where regular full compound RTCP transmission is
controlled by trr-int as described in {{RFC4585}}.

# Discussion {#discussion}

This section covers a few discussion points.

* Clock drift: SCReAM/SCReAMv2 can suffer from the same issues with clock drift
  as is the case with LEDBAT {{RFC6817}}. However, Appendix A.2 in {{RFC6817}}
  describes ways to mitigate issues with clock drift. A clockdrift compensation
  method is also implemented in {{SCReAM-CPP-implementation}}. Furthermore, the
  SCReAM implementation resets base delay history when it is determined that
  clock drift becomes too large. This is achieved by reducing the target bitrate
  for a few RTTs.

* Clock skipping: The sender or receiver clock can occasionally skip. Handling
  of this is implemented in {{SCReAM-CPP-implementation}}.

* The target bitrate given by SCReAMv2 is the bitrate including the RTP and
  Forward Error Correction (FEC) overhead. The media encoder SHOULD take this
  overhead into account when the media bitrate is set. This means that the media
  coder bitrate SHOULD be computed as: media_rate = target_bitrate -
  rtp_plus_fec_overhead_bitrate It is not necessary to make a 100% perfect
  compensation for the overhead, as the SCReAM algorithm will inherently
  compensate for moderate errors. Under-compensating for the overhead has the
  effect of increasing jitter, while overcompensating will cause the bottleneck
  link to become underutilized.

* The link utilization with SCReAMv2 can be lower than 100%. There are several
  possible reasons to this:

  - Large variations in frame sizes: Large variations in frame size makes
    SCReAMv2 push down the target_bitrate to give sufficient headroom and avoid
    queue buildup in the network. It is in general recommended to operate video
    coders in low latency mode and enable GDR (Gradual Decoding Refresh) if
    possible to minimize frame size variations.

  - Link layer properties: Media transport in 5G in uplink typically requires to
    transmit a scheduling request (SR) to get persmission to transmit
    data. Because transmission of video is frame based, there is a high
    likelihood that the channel becomes idle between frames (especially with
    L4S), in which case a new SR/grant exchange is needed. This potentially
    means that uplink transmission slots are unused with a lower link
    utilization as a result.

* Packet pacing is recommended, it is however possible to operate SCReAMv2 with
  packet pacing disabled. The code in {{SCReAM-CPP-implementation}} implements
  additonal mechanisms to achieve a high link utilization when packet pacing is
  disabled.

* RFC8888 Feedback issues: RTCP feedback packets can be lost, this means that
  the loss detection in SCReAMv2 may trigger even though packets arrive safely
  on the receiver side. {{SCReAM-CPP-implementation}} solves this by using
  overlapping RTCP feedback, i.e RTCP feedback is transmitted no more seldom
  than every 16th packet, and where each RTCP feedback spans the last 32
  received packets. This however creates unnecessary overhead. {{RFC3550}} RR
  (Receiver Reports) can possibly be another solution to achieve better
  robustness with less overhead.

# IANA Considerations {#iana}

This document does not require any IANA actions.

# Security Considerations {#security-considerations}

The feedback can be vulnerable to attacks similar to those that can affect
TCP. It is therefore RECOMMENDED that the RTCP feedback is at least integrity
protected. Furthermore, as SCReAM/SCReAMv2 is self-clocked, a malicious
middlebox can drop RTCP feedback packets and thus cause the self-clocking to
stall. However, this attack is mitigated by the minimum send rate maintained by
SCReAM/SCReAMv2 when no feedback is received.

--- back

# Acknowledgments {#acknowledgements}

Zaheduzzaman Sarker was a co-author of RFC 8298 the previous version
of scream which this document was based on. We would like to thank the
following people for their comments, questions, and support during the
work that led to this memo: Mirja Kuehlewind.
