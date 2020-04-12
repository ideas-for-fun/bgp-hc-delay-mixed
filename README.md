# bgp-hc-delay-mixed
Some ideas for building inter-AS TE-LSPs using BGP exchanges and mechanisms for Hop-count and/or delay


OPEN-SOURCE-IDEA                                           Balaji Venkat V
Intended Status: Experimental                  Ganesh Chennimalai Sankaran
                                                          Bhargav Bhikkaji
                                                  Indian Open Source Coop.
                                                   
                                                              
      Building Inter-AS TE-LSPs with HopCount and Delay using PCE 
                draft-balaji-idr-cspf-hc-delay-input-00


Abstract

   In order to build inter-AS TE-LSPs (Traffic Engineered Label Switched
   Paths) the existing methods use Path Computation Elements or PCEs,
   either as a single entity or as a federation of such PCEs. For
   constructing these inter-AS TE-LSPs it is necessary that the topology
   of the internet or a partial topology of the immediate neighborhood
   of the internet is necessary for the PCE. The source and the
   destination of the inter-AS TE LSP will delimit this partial topology
   with respect to the number of ASes to be covered. Currently the
   topology is derived from the BGP update protocol data units that are
   exchanged amongst the routers in the specified ASes under
   consideration. The essential data to be used to derive the topology
   from includes AS-PATH information in earlier methods prior to the
   introduction of BGP-LS (BGP Link state), and with the introduction of
   BGP-LS the specific updates with respect to BGP-LS which can be used
   to derive the said topology. Once this topology is derived and a
   graph of the ASes under consideration is constructed a suitable path
   that threads through specific ASes selected by the Constrained
   Shortest Path First algorithm on the PCE is derived and suitably the
   Explicit Route of the list of ASes is passed onto the PCC (Path
   computation client). This list of ASes is used to construct the
   inter-AS TE LSP using signalling protocols such as RSVP-TE. 

   The above is the existing state of the art. What is missing is that
   the internal topology of any single AS in the path contributes little
   to the decision making in the CSPF algorithm.  Unless the IGP Link
   state is also exported to the PCE the AS-PATH derived by the PCE can
   be sub-optimal with respect to the number of router hops that may
   exist even though the AS-PATH may be the shortest in length. Hence
   such a AS-PATH that does not take into consideration metrics like hop
   count and delay may contribute to delay characteristics that are
   undesirable for the inter-AS TE-LSP to be constructed. In order to
   get a path that is optimal in hop-count and delay as well as other
   constraints dictated by the PCC, the entire link state information of
   the IGP running in all the ASes (be it the entire internet or the
   partial set of ASes in the vicinity in a suitable diameter) needs to
 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 1]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   be looked, at considered and used for computation to derive such a
   optimality.

   The invention outlined in this document, suggests a method by which
   the entire IGP link state information is obviated and is stated as
   not required for such a computation. Using a set of new BGP update
   PDUs that calculate the hop-count and delay between ASBRs or EBGP
   speaking edge routers of the ASes under consideration a reformed CSPF
   is advocated so that the inter-AS TE-LSPs constructed as a result are
   optimal with respect to actual router hop-count and delay even if the
   AS-PATH so constructed is larger than the shortest AS-PATH available.

Balaji Venkat et.al       Expires May 26, 2020                  [Page 2]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019



Table of Contents

   1  Introduction  . . . . . . . . . . . . . . . . . . . . . . . . .  4
     1.1 Existing state of the art :  . . . . . . . . . . . . . . . .  5
     1.2  Terminology . . . . . . . . . . . . . . . . . . . . . . . .  5
   2.  Methodology of the Invention . . . . . . . . . . . . . . . . .  6
     2.2 Algorithm atop each EBGP Speaker / Autonomous System
         Border Router  . . . . . . . . . . . . . . . . . . . . . . .  7
       2.2.1 Rate Limiting the BGP PDU emanation based on HopCount 
             and Delay metric changes . . . . . . . . . . . . . . . .  9
         2.2.1.1 Directing the Computed HopCount and Delay PDUs to 
                 the Global PCE . . . . . . . . . . . . . . . . . . . 10
         2.2.1.2 Directing the Computed HopCount and Delay PDUs to 
                 the Hierarchical PCE in the vicinity . . . . . . . . 11
       2.2.2 Constrained Shortest Path based on HopCount and Delay  . 13
         2.2.2.1 Algorithm 1 ASBR(j) HopCount and Delay Computation
                 Mechanism  . . . . . . . . . . . . . . . . . . . . . 13
         2.2.2.2 Algorithm 2 PCE's Inter-AS TE-LSP based on
                 HopCount and Delay . . . . . . . . . . . . . . . . . 14
         2.2.2.3 Explicit routing using TE-LSPs . . . . . . . . . . . 14
     2.3 Equivalence class with total ordering  . . . . . . . . . . . 15
       2.3.1 Algorithm 3 PCE low-power path algorithm with graph 
             labeling . . . . . . . . . . . . . . . . . . . . . . . . 16
     2.4 BGP-LinkState Hop Count Attribute Protocol Data Unit . . . . 18
     2.5 BGP-LinkState Delay Attribute Protocol Data Unit . . . . . . 19
   3  Security Considerations . . . . . . . . . . . . . . . . . . . . 20
   4  IANA Considerations . . . . . . . . . . . . . . . . . . . . . . 20
   5  References  . . . . . . . . . . . . . . . . . . . . . . . . . . 20
     5.1  Normative References  . . . . . . . . . . . . . . . . . . . 20
     5.2  Informative References  . . . . . . . . . . . . . . . . . . 20
   Authors' Addresses . . . . . . . . . . . . . . . . . . . . . . . . 21












 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 3]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


1  Introduction

   In order to build inter-AS TE-LSPs (Traffic Engineered Label Switched
   Paths) the existing methods use Path Computation Elements or PCEs,
   either as a single entity or as a federation of such PCEs. For
   constructing these inter-AS TE-LSPs it is necessary that the topology
   of the internet or a partial topology of the immediate neighborhood
   of the internet is necessary for the PCE. The source and the
   destination of the inter-AS TE LSP will delimit this partial topology
   with respect to the number of ASes to be covered. Currently the
   topology is derived from the BGP update protocol data units that are
   exchanged amongst the routers in the specified ASes under
   consideration. The essential data to be used to derive the topology
   from includes AS-PATH information in earlier methods prior to the
   introduction of BGP-LS (BGP Link state), and with the introduction of
   BGP-LS the specific updates with respect to BGP-LS which can be used
   to derive the said topology. Once this topology is derived and a
   graph of the ASes under consideration is constructed a suitable path
   that threads through specific ASes selected by the Constrained
   Shortest Path First algorithm on the PCE is derived and suitably the
   Explicit Route of the list of ASes is passed onto the PCC (Path
   computation client). This list of ASes is used to construct the
   inter-AS TE LSP using signalling protocols such as RSVP-TE. 

   The above is the existing state of the art. What is missing is that
   the internal topology of any single AS in the path contributes little
   to the decision making in the CSPF algorithm.  Unless the IGP Link
   state is also exported to the PCE the AS-PATH derived by the PCE can
   be sub-optimal with respect to the number of router hops that may
   exist even though the AS-PATH may be the shortest in length. Hence
   such a AS-PATH that does not take into consideration metrics like hop
   count and delay may contribute to delay characteristics that are
   undesirable for the inter-AS TE-LSP to be constructed. In order to
   get a path that is optimal in hop-count and delay as well as other
   constraints dictated by the PCC, the entire link state information of
   the IGP running in all the ASes (be it the entire internet or the
   partial set of ASes in the vicinity in a suitable diameter) needs to
   be looked, at considered and used for computation to derive such a
   optimality.

   The invention outlined in this document, suggests a method by which
   the entire IGP link state information is obviated and is stated as
   not required for such a computation. Using a set of new BGP update
   PDUs that calculate the hop-count and delay between ASBRs or EBGP
   speaking edge routers of the ASes under consideration a reformed CSPF
   is advocated so that the inter-AS TE-LSPs constructed as a result are
   optimal with respect to actual router hop-count and delay even if the
   AS-PATH so constructed is larger than the shortest AS-PATH available.
 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 4]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


1.1 Existing state of the art :

   Consider the following AS topology in the near vicinity of AS 100.
   There exists a source S that is attached to AS 100 and a desintation
   D that is attached to AS 600. If this topology were to be gathered by
   tapping into the BGP peering with one or more ASes by the PCE in the
   vicnity then the shortest AS PATH from S to D would be through AS
   100, 300 and 600 in that order. Assume that the hop counts and delay
   metrics between the ingress and egress routers within each of the
   ASes is not considered, the AS-PATH length would dictate that the
   shortest AS-PATH be chosen hence the path 100,300,600 in that order.
   Consider this that the hop count of routers in the path AS
   100,300,400,500 is much lesser than the AS-PATH 100,300,600. Under
   the constraint that the shortest router hop-count path be chosen the
   path through 400,500 should get greater favourable treatment from the
   PCE. Hence the knowledge of the hop-count and delay on the specific
   paths within each AS that make up the segments of the inter-AS path
   is an important feed into the CSPF computation at the PCE. The
   invention proposed in this document renders that very knowledge using
   a few additional BGP protocol data units in addition to the other
   attributes of the topology constructed.


      +--------+       +--------+                       +--------+
   S--| AS 100 |-------| AS 300 |-----------------------| AS 600 |-- D
      +--------+       +--------+                       +--------+
          |                |                               |
          |                |                               |
          |                |                               |
      +--------+       +--------+         +--------+       |
      | AS 200 |-------| AS 400 |---------| AS 500 |-------+
      +--------+       +--------+         +--------+

   Figure 1.0 Illustrative Topology 


1.2  Terminology

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in RFC 2119 [RFC2119].







 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 5]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   AS - Autonomous System
   PCC - Path Computation Client
   PCE - Path Computation Element
   PCEP - Path Computation Element Protocol
   ASBR(j) - Ingress ASBR router in AS(j)
   N(i) - Egress ASBR router in AS(j)
   AS(D) - Downstream AS for traffic
   AS(U) - Upstream AS for traffic
   TE - Traffic Engineering
   TE-LSP - Traffic Engineered Label Switched Path
   MPLS - Multi-protocol Label switching
   SR - Segment Routing

2.  Methodology of the Invention

   The intent of the invention is to calculate the following metrics

   a) HopCount

   This denotes the number of router hops between the ingress ASBR and
   the egress ASBR in an AS where the egress ASBR connects the said AS
   to the traffic-downstream AS in the AS-PATH to be followed by the
   packets in the inter-AS TE-LSP.

   b) Delay

   This denotes the delay between the ingress ASBR and the egress ASBR
   in an AS where the egress ASBR connects the said AS to the traffic-
   downstream AS in the AS-PATH to be followed by the packets in the
   inter-AS TE-LSP.

   These set of metrics are calculated for every combination of ingress
   and egress ASBRs in an AS and thus within every Autonomous system
   where the mechanism in the invention is deployed.

   Using the metrics so calculated a Path Computation element is given
   this set of information from every AS participating in the scheme.
   The PCE then uses these metrics to calculate an inter-AS path either
   in whole or in sections (by participating federation of PCEs) by
   choosing that inter-AS path that is least in terms of HopCount from
   source to destination and/or least in terms of delay from source to
   destination.

   A set of algorithms that implement this scheme are specified in the
   below sections.

   It is to be understood that if the PCC or Path Computation Client
   requests the PCE in question to compute the paths based on the
 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 6]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   HopCount and Delay metrics then the PCE in question adheres to the
   following algorithms. It is up to the PCC as per its configuration to
   dictate that the PCE compute the Inter-AS TE-LSP based on the
   HopCount and Delay metrics if they are available.

2.2 Algorithm atop each EBGP Speaker / Autonomous System Border Router

   Step 1) The said Speaker ASBR(j) would first Determine the set of
   External Autonomous Systems to which the said EBGP Speaker is
   connected to. This list is to be named List-External-AS.


   Step 2) Determine the set of other EBGP Speakers in the same
   Autonomous System. This list is to be named List-Other-EBGP-Speakers-
   in-AS.

   Step 3) For every node N(i) in List-Other-EBGP-Speaker-in-AS
   calculate hop-count and delay metrics as follows...

   a) Said Speaker sends OAM messages using a traceroute like mechanism
   to determine the hop-count from itself to N(i). It is possible that
   the OAM messages through traceroute are sent using the IGP best-path
   metrics or through a specific traceroute implementation that follows
   a path over a Traffic Engineered Tunnel already established with a
   set of characteristics like delay and bandwidth in mind. It is upto
   the Said Speaker ASBR(j) to decide how it would prefer to obtain the
   hop-count either through the regular IGP metric to N(i) and/or
   through a set of already established tunnels to N(i). ASBR(j) then
   records the hop-count to every neighbor N(i) has been obtained either
   through the IGP metric method or through an already established TE-
   Tunnel. Here the regular traceroute or MPLS LSP Traceroute could be
   used to determine the HopCount metric as the case may be.

   b) Said Speaker ASBR(j) sends OAM messages using a traceroute like
   mechanism as in (a) to determine the delay from itself to N(i). It is
   possible that the OAM messages through traceroute are sent using the
   IGP best-path metrics or through a specific traceroute implementation
   that follows a path over a Traffic Engineered Tunnel already
   established with a set of characteristics like delay and bandwidth in
   mind. It is upto the Said Speaker ASBR(j) to decide how it would
   prefer to obtain the delay either through the regular IGP metric to
   N(i) and/or through a set of already established tunnels to N(i).
   ASBR(j) then records the delay to every neighbor N(i) has been
   obtaied either through the IGP metric method or through an already
   established TE-Tunnel. Here the regular traceroute or Ping or MPLS
   LSP Ping could be used to determine the Delay metric as the case may
   be.

 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 7]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   Please note that in both (a) and (b) it is possible that both options
   are used to calculate the HopCount and Delay metrics and are sent in
   a TLV format in the BGP update using the PDUs that are outlined in
   the section below. If there exist more than one AS-internal TE-LSP or
   IGP shortest paths between ASBR(j) and N(i) then the appropriate TLVs
   in the BGP update will record all such possible paths and the
   HopCount and Delay metrics for such paths thereof.

   c) Said Speaker ASBR(j) then constructs the following Protocol Data
   units for updating its External AS neighbor EBGP speakers for
   denoting the hop-count and delay to N(i) through which the said AS of
   ASBR(j) is connected to its traffic-downstream External AS. The
   values would be collated as HopCount-to-N(i) and Delay-to-N(i) to get
   to traffic-downstream External AS(D) to which N(i) is connected to.

   d) This constructed set of PDUs may be directly sent to a
   Hierarchical Path Computation Element PCE or to the traffic-upstream
   External AS(U) to which said EBGP speaker ASBR(j) is connected to.
   The former method adopted will use the data as local information to
   the Hierarchical PCE in question, while the latter method will use
   what will be called the Global Method of PCE CSPF computation.

   IMPORTANT NOTE : By default every N(i) will be considered a ASBR(j).



                          .____________.
                         (              )
     (Upstream AS 300)  (                )
         \             (       +--------->(N(1)) ...>(ASBR(1) of AS 200)
          \        +-------+  / HopCount = 4                 ^
           +------>|       | /                               .
                   |ASBR(1)|------------>(N(2))...............
           +------>|  of   |\  HopCount = 5       (ASBR(2) of AS 200)
          /        | AS 100| \                                ^
         /         +-------+  \                               .
     (Upstream AS 400) (      +-------->(N(3))  ...............
                        (      HopCount = 7
                         ._____________.

   Figure 1.1 : ASBR(j) and its relationship with traffic-downstream
   Egress Routers N(i) HopCount-wise






 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 8]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


                          .____________.
                         (              )
     (Upstream AS 300)  (                )
         \             (     +--------->(N(1)) ....>(ASBR(1) of AS 200)
          \        +-------+ / Delay = 0.4                  ^
           +------>|       |/                               .
                   |ASBR(1)|------------>(N(2))..............
           +------>|  of   |\  Delay = 0.3          (ASBR(2) of AS 200)
          /        | AS 100| \                                ^
         /         +-------+  \                               .
     (Upstream AS 400) (      +-------->(N(3)) ................
                        (      Delay = 0.7   
                         ._____________.

   Figure 1.2 : ASBR(j) and its relationship with traffic-downstream
   Egress Routers N(i) Delay-wise

2.2.1 Rate Limiting the BGP PDU emanation based on HopCount and Delay
   metric changes

   Frequent emanation of the HopCount and Delay metric is to be avoided
   as this means constant chatter in the BGP connection between the said
   AS and its traffic-upstream neighbors. A mechanism that does the
   following would lessen the constant chatter if deemed unnecessary. 

   The OAM measurements are periodically done with a suitably configured
   time interval on the ASBR(j) in question. Suitable thresholds are
   configured in terms of the delta between the current OAM measurement
   for HopCount and Delay and its previous measurement. If the threshold
   is NOT exceeded then the BGP PDU is NOT emanated. If the threshold
   PLUS a certain buffer is exceeded (where the buffer in HopCount and
   Delay is configurable) then it will be found suitable to emanate the
   BGP PDU. 

2.2.1.0.1 When links and nodes go down within the said AS

   When links and/or nodes (routers) go down within the said AS then the
   paths are bound to change between the ASBR(j) and the N(i) EBGP
   speaking neighbor. This may result in a change in the measured
   HopCount and Delay. In such a case the threshold configured in
   section 2.2.1.0 will dictate whether a new BGP PDU is to be emanated
   or not.

2.2.1.0.2 When the ASBR(j) or the N(i) goes down

   When the ingress ASBR(j) or the egress N(i) itself are subject to
   failure, it is given that the local Fast Reroute mechanisms such as
   MPLS-FRR or Segment Routing based FRR mechanisms will take over and
 


Balaji Venkat et.al       Expires May 26, 2020                  [Page 9]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   conquer whatever failure that occurs in that section of the inter-AS
   TE-LSP. In any case the PCE will get to know of such a failure and
   would have computed a backup path onto which packets can be
   redirected or will let local FRR mechanisms to take over while it
   calculates a new backup path with that ingress and/or egress ASBR
   being taken out of the equation in the new backup path calculation.

   When the inter-AS TE-LSP faces a failure it is usual that the head-
   end is notified of such a failure either in the global source or at
   some section of the inter-AS TE-LSP path. This will cause the PCC to
   raise a request with the PCE (global or Hierarchical PCE) to compute
   a new path while the local FRR mechanisms have taken over in the
   vicinity of the failure.

   Subsequent to a ASBR(j) or the egress N(i) going down or links to
   these respective entities going down, the hop-count or delay would be
   set to an infinity value appropriately which will result in the said
   BGP-LinkState Updates (prior to the links or nodes going down) to be
   discarded or set to an infinity value. This would cause all TE-LSPs
   primary or backup, set up after such failure to avoid the said nodes
   that have gone down since the infinity value would obviate the need
   for using the said node in the topology constructed by the PCE, of
   the said Autonomous System.


2.2.1.1 Directing the Computed HopCount and Delay PDUs to the Global PCE

   The so computed BGP PDUs containing the HopCount and Delay metrics
   from ASBR(j) to every N(i) and thus to every AS(D) which are traffic-
   downstream, are sent to every corresponding AS(U) which are traffic-
   upstream from ASBR(j) in the Global PCE based method. These BGP PDUs
   are then passed on to the upstream ASes of AS(U) as well. Thus the
   messages travel to the PCE in the vicinity that performs CSPF
   computation for the set of ASes in the grouping in the vicinity in
   question.

   The PCE then collects these PDUs and begins constructing the Inter-AS
   Internet Topology of the vicinity in question. A graph is built from
   the BGP-LinkState Updates and links between the ASBR(j) and the N(i)
   downstream ASBR's of every AS in question in the topology is assigned
   the HopCount and/or Delay metrics so collected. Subsequently the
   algorithm in the following section 2.2.1 is computed and the
   resultant AS-PATH so constructed is passed onto the head-end router
   of the Inter-AS TE-LSP and the path so specified is constructed using
   RSVP-TE. Alternately the Node-IDs of the intermediate routers can be
   used in the Segment Routing context to build a stack of labels for
   the Inter-AS TE-LSP.

 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 10]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   With respect to Segment Routing it is important to note that the
   interconnection between ASBR(j) and its specific N(i) choice could be
   designated with Node-ID SR based prefix with an associated MPLS label
   to indicate the choice of N(i) to be taken to exit to the AS(D) in
   question in the traffic-downstream direction. This label is also
   included in the BGP PDU sent to the PCE in the vicinity.

   If the HopCount or Delay metric was annotated with the IGP metric
   then the Inter-AS TE-LSP so constructed follows the IGP shortest path
   in question between the ASBR(j) and N(i) neighbor for the AS in
   question. This is purely a local decision for the ASBR(j) to take up
   when forwarding the packets in that section of the path to its
   neighbor N(i).

   If the HopCount or Delay metric was annotated with an already
   existing TE-LSP-ID of a AS-internal TE-LSP in question, then again it
   is a local decision for the ASBR(j) to make the packets pass through
   the conduit of the TE-LSP whose TE-LSP-ID is mentioned in the BGP
   update that conveyed the information in the first place.

2.2.1.2 Directing the Computed HopCount and Delay PDUs to the
   Hierarchical PCE in the vicinity

   In this local PCE method the so computed BGP PDUs containing the
   HopCount and Delay metrics from ASBR(j) to every N(i) and thus to
   every AS(D) which are traffic-downstream, are sent to the
   Hierarchical PCE in the vicinity that is in charge of CSPF
   computation for the AS or a set of ASes that are handled by the PCE.

   In such a case the BGP PDUs for HopCount and Delay are not meant to
   travel through the entire Internet or to a Global PCE in question,
   rather it would travel merely to the PCE that has been delegated the
   responsibility of computing TE-LSPs that involve the AS or the set of
   ASes that are being handled by the PCE.

   The PCE then collects these PDUs and begins constructing the Inter-AS
   local Topology of the vicinity in question. A graph is built from the
   BGP-LinkState Updates and links between the ASBR(j) and the N(i)
   downstream ASBR's of every AS in question in the topology is assigned
   the HopCount and/or Delay metrics so collected. Subsequently the
   algorithm in the following section 2.2.1 is computed and the
   resultant AS-PATH so constructed is passed onto the head-end router
   of the Inter-AS TE-LSP and the path so specified is constructed using
   RSVP-TE. Alternately the Node-IDs of the intermediate routers can be
   used in the Segment Routing context to build a stack of labels for
   the Inter-AS TE-LSP.

   With respect to Segment Routing it is important to note that the
 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 11]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   interconnection between ASBR(j) and its specific N(i) choice could be
   designated with Node-ID SR based prefix with an associated MPLS label
   to indicate the choice of N(i) to be taken to exit to the AS(D) in
   question in the traffic-downstream direction. This label is also
   included in the BGP PDU sent to the PCE in the vicinity.

   If the HopCount or Delay metric was annotated with the IGP metric
   then the Inter-AS TE-LSP so constructed follows the IGP shortest path
   in question between the ASBR(j) and N(i) neighbor for the AS in
   question. This is purely a local decision for the ASBR(j) to take up
   when setting up the inter-AS TE-LSP sub-section between itself and
   the chosen N(i) in the path to be used for forwarding the packets its
   selected EBGP speaking neighbor N(i) towards the destination.

   If the HopCount or Delay metric was annotated with an already
   existing TE-LSP-ID of a AS-internal TE-LSP in question, then again it
   is a local decision for the ASBR(j) to to take up when setting up the
   inter-AS TE-LSP sub-section between itself and the chosen N(i) in the
   path to be used for forwarding the packets for that LSP. This is done
   in order to make the packets pass through the conduit of the already
   existing TE-LSP whose TE-LSP-ID is mentioned in the BGP update that
   conveyed the information in the first place.

2.2.1.2.1 For Autonomous Systems not implementing this mechanism

   For ASes that do not implement this specific mechanism of calculating
   the HopCount and Delay between each ASBR(j) to its N(i) EBGP speaking
   neighbor, the BGP PDUs are not emanated to the Global or the
   Hierarchical PCE in question. In such a case the PCE will search for
   paths that contain the information and if there exists no next-hop AS
   in all the AS-PATHs to the destination an appropriate AS next-hop is
   chosen that lies proximal to other downstream ASes that do contain
   this information. Appropriate heuristics can be followed that choose
   a path for the inter-AS TE-LSP that contains sufficient number of
   ASes that do contain the HopCount and Delay information if these
   metrics are to be applied. 

   If however there is paucity of such information regarding HopCount
   and Delay metrics in the said cluster of ASes between the source and
   destination of the Inter-AS TE-LSP to be computed then the other
   legacy methods of computation SHOULD be followed.

   If however the majority of the ASes in the vicinity are found to be
   deploying this mechanism then the PCE is bound and SHOULD compute the
   path based on these said metrics namely HopCount and Delay if the
   said method of computation is requested by the Path Computation
   Client or PCC.

 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 12]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


2.2.2 Constrained Shortest Path based on HopCount and Delay

   On each Ingress ASBR, ASBR(j) in the AS concerned, the following
   computation takes place...

2.2.2.1 Algorithm 1 ASBR(j) HopCount and Delay Computation Mechanism

    Require: Weighted Topology Graph T=(AS, E, f)
    1: Begin
    2: if ROUTER == ASBR(j) then
    3: /* As part of HopCount and Delay Calculation */
    4: Trigger exchange of OAM messages to the other AS 
       internal EBGP Speakers/neighbors in set N(i) periodically
       which is based on Algorithm in section 2.2.
    5: BEGIN PARALLEL PROCESS 1
    6: while Hop Count and Delay changes happen do
    7: Assign the HopCount and Delay metrics to the 
       Ingress links to traffic-upstream ASes AS(U);
    8: Exchange the HopCount and Delay metrics with its 
       external neighbors which are traffic-upstream ASes AS(U)
       which includes annotating the metric update whether it
       it was decided based on IGP-Metric or TE-LSP-ID if done
       through a specific TE-Tunnel already established either
       through RSVP-TE or through LDP or through any other means.
    10: end while
    11: END PARALLEL PROCESS 1
    12: BEGIN PARALLEL PROCESS 2
    13: while RSVP packets arrive do
    14: Send and Receive TE-LSP reservations in the explicit path;
    15: Update routing table with labels for TE-LSP;
    16: end while
    17: END PARALLEL PROCESS 2
    18: end if
    19: End


   IMPORTANT NOTE : By default every N(i) is considered to be part of
   the set ASBR(j).










 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 13]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   So there exist multiple ways of deriving the CSPF path at the
   Hierarchical PCE or in a Global PCE.

   If at the time of constructing the inter-AS TE-LSP using the HopCount
   and Delay metrics so computed in step (c) of the Algorithm in section
   2.2, the BGP update PDUs so constructed are passed using the Global
   Method or the Local Method, the appropriate PCE tapping into the BGP-
   LinkState updates will use the HopCount and/or Delay metrics as
   follows...

2.2.2.2 Algorithm 2 PCE's Inter-AS TE-LSP based on HopCount and Delay
    Require: Weighted Topology Graph T=(AS, E, f)
    Require: Source and Destination for Inter-AS TE LSP with sufficient
   bandwidth and least hop-count and/or least delay.
    1: Begin
    2: if ROUTER == PCE then
    3: Calculate the shortest paths from the head-end to the
       tail-end using CSPF with SUMOF(HopCount) AND/OR SUMOF(Delay)
       as the metric with sufficient bandwidth to carry the intended
   traffic;
       Here the HopCount of every pair of ASBR(j) and N(i) within
       each AS in the AS-PATHs available to the destination are summed
       up and used as the metric, the least SUM of which will indicate
       the chosen AS-PATH through which the inter-AS TE-LSP is to carry
       the packets from the source to the destination;
    4: if no path available then
    5: Signal error;
    6: end if
    7: if path exists then
    8: Send explicit path to head-end to construct path;
    9: end if
    10: Continue passively listening to BGP updates to update
        T=(AS, E, f);
    11: end if
    12: End

2.2.2.3 Explicit routing using TE-LSPs

   We assume that the head-end and the tail-end may reside in different
   AS and the path is along multiple intervening AS. The way to generate
   this path is by using Traffic Engineered Label Switched Paths (TE-
   LSPs). TE-LSPs can influence the exact path (at the AS level) that
   the traffic will pass through. This path can then be realized by
   providing these set of low-power consuming AS to a protocol like
   Resource Reservation Protocol (RSVP). RSVP-TE then creates TE-LSPs or
   tunnels, using its label assigning procedure. The routers use these
   low-power paths created by the explicit routing method rather than
   using the conventional shortest path to the destination. By this way,
 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 14]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   we can influence exclusion of a number of high power AS on the way
   from the head-end to the tail-end AS. For example, the dotted line in
   Figure 2.0 represents the explicit route that is chosen by making use
   of such TE-LSPs from head-end AS A to the tail-end AS X. Note that if
   number of AS hops was the metric used by CSPF, then the route chosen
   is the path with 3 hops.


                       0.5
             (C)  +----------------+
           0.5|  /                 |
              | /                  |
         0.05 V/ 0.1   0.03   0.2  V
      (A)--->(B)--->(D)--->(G)--->(H)
              |             |      |
              |          0.5|      | 0.1
              |             V      V
              +----------->(E)--->(X)
                 0.5           0.3

   Figure 2.0:Low-delay path is represented by the dotted lines. This
   low- delay path has a longer number of AS hops than the conventional
   shortest path.



                       5
             (C)  +----------------+
          15  |  /                 |
              | /                  |
          5   V/   1      3     2  V
      (A)--->(B)--->(D)--->(G)--->(H)
              |             |      |
              |            8|      |   1
              |             V      V
              +----------->(E)--->(X)
                   9             3

   Figure 3.0:Low-HopCount path is represented by the dotted lines. This
   low- HopCount path has a longer number of AS hops than the
   conventional shortest path.

2.3 Equivalence class with total ordering

   The heuristic is based on avoiding links between ASes that are High
   on HopCount and/or Delay metrics. The approach partitions the
   weighted links into equivalence classes based on a range of HopCount
   and/or Delay values. For each partition a labeling is applied such
 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 15]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


   that each link in the partition has the same label. A total ordering
   relationship is then defined on the equivalence class. The heuristic
   then starts including partitions with minimum label value iteratively
   until we get a connected component, which includes the head-end and
   tail-end AS. We apply the CSPF algorithm with the weights as label
   values on this sub-graph to obtain the Least HopCount and/or Delay
   paths. The modified algorithm which uses this scheme is given in
   Algorithm 3. It should be noted that this algorithm could provide
   sub-optimal paths as the intermediate steps carry incomplete Internet
   topology information.  


2.3.1 Algorithm 3 PCE low-power path algorithm with graph labeling

    Require: Weighted Topology Graph T=(AS, E, f)
    Require: Source and Destination for Inter-AS TE LSP with
             sufficient bandwidth and least hop-count and/or delay.
    1: Begin
    2: if ROUTER == PCE then
    3: Group the links into N partitions with a label for
       each partition depending on the HopCount and/or Delay metric 
       coupled with required bandwidth.
    4: Sort the labels in ascending order.
    5: repeat
    6: Include the links that have the least label value;
    7: Remove the partition with this label;
    8: until there is a path from the head-end to tail-end AS
    9: Calculate the Least HopCount and/or Delay metric path using
   labels from the
       head-end to the tail-end using CSPF ;
    10: if no path available then
    11: Signal error;
    12: end if
    13: if path exists then
    14: Send explicit path to head-end to construct path;
    15: end if
    16: Continue passively listening to BGP updates to
        update T=(AS, E);
    17: end if
    18: End








 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 16]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


                    +------(1245)-----+
                  /      / Y            \
                 V      /           Y    \
              (1339)   +   (6785)<------(1377)----+
                |      |     |           /        |
                | G    |   G |          + G       V R
                |      |     V          V      (2485)
                V      |   (1006)    (3426)--+    |
              (34234)  |   Y |      G  |     |    | R
                |      |     V         V     |    V
                | G    |   (11229)   (4563)  | (5677)
                V      |   Y |         |     V     | R
              (23411)  |     V     Y   +-->(7786)<-+
                |      +-->(1485)<--------/  |
                |          Y |               | R
                |            V               V
                |          (15467)        (8298)
                | G        G |               | R
                |            |               |
                |            V               V
                +--------->(16578)<-------(9732)

   Figure 4.0:Application of the graph-labeling heuristic. We consider 3
   labels "G" < "Y" < "R". Using algorithm 3 the "G" path from the head-
   end AS 1245 to the tail-end AS 16578 is chosen in the first
   iteration.






















 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 17]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


2.4 BGP-LinkState Hop Count Attribute Protocol Data Unit

        0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
        +-------------------------------------------------------------+
        |          Owning 32 bit Autonomous System Number             |
        +-------------------------------------------------------------+
        |          Other  32 bit Autonomous System Number             |
        +-------------------------------------------------------------+
        |          Advertising ASBR's IP router ID                    |
        +-------------------------------------------------------------+
        |              Peer ASBR's IP router ID                       |
        +-------------------------------------------------------------+
        |          Variable Portion consisting of multiple TLVs       |
        |          ................................................   |
        |          Number of TLVs encoded |    Opcode for HopCount    |
        +-------------------------------------------------------------+
        | Next hop N(i), Downstream AS Number, HopCount, IGP or TE-LSP|
        +-------------------------------------------------------------+
        | TE-LSP-ID if TE-LSP selected                                |
        ~                                                             ~
        | TLVs Repeated as many number of times as in Number encoded  |
        +-------------------------------------------------------------+

   Figure 5.0: Proposed PDU format for HopCount metric between ASBR(j)
   and every N(i) to every traffic-downstream AS(D)























 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 18]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


2.5 BGP-LinkState Delay Attribute Protocol Data Unit

        0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
        +-------------------------------------------------------------+
        |          Owning 32 bit Autonomous System Number             |
        +-------------------------------------------------------------+
        |          Other  32 bit Autonomous System Number             |
        +-------------------------------------------------------------+
        |          Advertising ASBR's IP router ID                    |
        +-------------------------------------------------------------+
        |              Peer ASBR's IP router ID                       |
        +-------------------------------------------------------------+
        |          Variable Portion consisting of multiple TLVs       |
        |          ................................................   |
        |          Number of TLVs encoded |    Opcode for Delay       |
        +-------------------------------------------------------------+
        | Next hop N(i), Downstream AS Number, Delay, IGP or TE-LSP   |
        +-------------------------------------------------------------+
        | TE-LSP-ID if TE-LSP selected                                |
        ~                                                             ~
        | TLVs Repeated as many number of times as in Number encoded  |
        +-------------------------------------------------------------+

   Figure 6.0: Proposed PDU format for Delay metric between ASBR(j) and
   every N(i) to every traffic-downstream AS(D)























 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 19]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


3  Security Considerations

   Appropriate authentication of the messages in a BGP connection either
   between the ASBR(j) in question and its traffic-upstream EBGP
   neighbor in a different AS is done using already existing state of
   the art mechanisms in place. The same could be said of the connection
   between the ASBR(j) and the local PCE that is delegated the task of
   computing the CSPF for that AS or the set of ASes in the vicinity.

   No additional security measures are to be followed as the existing
   mechanisms suffice.


4  IANA Considerations

   For the BGP-LinkState PDUs in question, a suitable assignment from
   IANA would be necessary in the appropriate registry of numbers and
   codes to denote the HopCount and Delay Attributes of the said link
   between each ASBR(j) in the AS and its corresponding N(i) linkage.

   Also the PCEP protocol that is on-going between a PCC and the PCE
   would require additions to its supported PDUs to let the PCC indicate
   to the PCE that a HopCount and/or Delay based inter-AS TE-LSP is to
   be constructed. This would let the PCE know that it is to compute the
   inter-AS TE-LSP based on the HopCount and/or Delay metrics as
   specified by the IANA code in the PDU from the PCC to the PCE.

5  References

5.1  Normative References

   [KEYWORDS] Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119, DOI
              10.17487/RFC2119, March 1997, <http://www.rfc-
              editor.org/info/rfc2119>.

   [RFC1776]  Crocker, S., "The Address is the Message", RFC 1776, DOI
              10.17487/RFC1776, April 1 1995, <http://www.rfc-
              editor.org/info/rfc1776>.

   [TRUTHS]   Callon, R., "The Twelve Networking Truths", RFC 1925, DOI
              10.17487/RFC1925, April 1 1996, <http://www.rfc-
              editor.org/info/rfc1925>.


5.2  Informative References

   [EVILBIT]  Bellovin, S., "The Security Flag in the IPv4 Header",
 


Balaji Venkat et.al       Expires May 26, 2020                 [Page 20]

OPEN-SOURCE-IDEA  Inter-AS TE-LSPs with HopCount and Delay24 November 2019


              RFC 3514, DOI 10.17487/RFC3514, April 1 2003,
              <http://www.rfc-editor.org/info/rfc3514>.

   [RFC5513]  Farrel, A., "IANA Considerations for Three Letter
              Acronyms", RFC 5513, DOI 10.17487/RFC5513, April 1 2009,
              <http://www.rfc-editor.org/info/rfc5513>.

   [RFC5514]  Vyncke, E., "IPv6 over Social Networks", RFC 5514, DOI
              10.17487/RFC5514, April 1 2009, <http://www.rfc-
              editor.org/info/rfc5514>.


Authors' Addresses


   Name
   and
   Address

   EMail: name@example.com


























Balaji Venkat et.al       Expires May 26, 2020                 [Page 21]
