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
