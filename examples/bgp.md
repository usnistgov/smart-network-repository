# BGP "what-if" scenario
Follow the steps below to run the example. 

1. Access the Neo4j console from a browser at http://localhost:7474. 
Use `neo4j` `12345-password` if prompted to connect.

2. Run the following queries in the prompt:
```bash
// declare an empty procedure to avoid recursivity error
CALL apoc.custom.declareProcedure("runBGP(route_id::INT) :: (ignored::INT)", "RETURN null AS ignored", 'read')
```
```bash
CALL apoc.custom.declareProcedure("runBGP(route_id::INT) :: (ignored::INT)", 
"WITH $route_id AS route_id
MATCH (r1:Router)-[:HAS_ROUTE]->(route:Route) WHERE id(route) = route_id
MATCH (r1)-[:HAS_BGP_FILTER]->(filter:BgpFilter {type: 'outbound'})
WITH r1, route, filter, net.ip.isInPrefixRange(filter.prefix, route.prefix) AS result WHERE result = true OR filter.filter_as = route.origin_as 
WITH r1, route, filter
ORDER BY filter.priority ASC
WITH r1, route, head(collect(filter)) as firstMatchingFilter
MERGE (firstMatchingFilter)-[:APPLIES_OUTBOUND_FILTER {action: firstMatchingFilter.action, simulation: true}]->(route)
WITH r1, route, firstMatchingFilter WHERE firstMatchingFilter.action = 'PERMIT'
MATCH (r1)-[:HAS_PORT]->(ip:Port)-[:HAS_BGP_PEER]->(bgpPeer:BgpPeer)
WHERE NOT (route)-[:LEARNED_FROM]->(:BgpUpdate)<-[:RECEIVES_UPDATE]-(bgpPeer)
MERGE (bgpUpdate:BgpUpdate {prefix: route.prefix, origin_as: route.origin_as, local_pref: route.local_pref, med: route.med, next_hop: ip.ipAddr, communities: route.communities, path: route.path + r1.asn})
MERGE (bgpPeer)-[:BGP_UPDATE_TO_SEND {simulation:true}]->(bgpUpdate)
SET bgpUpdate.simulation = true
WITH bgpPeer, bgpUpdate
MATCH (bgpPeer)-[:HAS_BGP_SESSION]-(bgpPeer2:BgpPeer)
MERGE (receivedUpdate:BgpUpdate {prefix: bgpUpdate.prefix, origin_as: bgpUpdate.origin_as, local_pref: bgpUpdate.local_pref, med: bgpUpdate.med, next_hop: bgpUpdate.next_hop, communities: bgpUpdate.communities, path: bgpUpdate.path, simulation: true})
MERGE (bgpPeer2)-[ru:RECEIVES_UPDATE {simulation: true}]->(receivedUpdate)
WITH bgpPeer2, receivedUpdate
MATCH (bgpPeer2)<-[:HAS_BGP_PEER]-(:Port)<-[:HAS_PORT]-(r2:Router)
WHERE NOT r2.asn IN receivedUpdate.path
MATCH (r2)-[:HAS_BGP_FILTER]->(filter:BgpFilter {type: 'inbound'})
WITH r2, receivedUpdate, filter, net.ip.isInPrefixRange(filter.prefix, receivedUpdate.prefix) AS result WHERE result = true OR filter.filter_as = receivedUpdate.origin_as
WITH r2, receivedUpdate, filter
ORDER BY filter.priority ASC
WITH r2, receivedUpdate, head(collect(filter)) as firstMatchingFilter
MERGE (firstMatchingFilter)-[:APPLIES_INBOUND_FILTER {action: firstMatchingFilter.action, simulation: true}]->(receivedUpdate)
WITH r2, receivedUpdate, firstMatchingFilter WHERE firstMatchingFilter.action = 'PERMIT'
CREATE (r2)-[:HAS_ROUTE {simulation: true}]->(received_route:Route {prefix: receivedUpdate.prefix, origin_as: receivedUpdate.origin_as})-[:LEARNED_FROM {simulation: true}]->(receivedUpdate)
SET received_route.med = receivedUpdate.med,
    received_route.next_hop = receivedUpdate.next_hop,
    received_route.path = receivedUpdate.path,
    received_route.simulation = true,
    received_route.originRouteId = $route_id,
    received_route.communities = CASE WHEN firstMatchingFilter.set_communities IS NOT NULL THEN receivedUpdate.communities + firstMatchingFilter.set_communities ELSE receivedUpdate.communities END,
    received_route.local_pref = CASE WHEN firstMatchingFilter.set_local_pref IS NOT NULL THEN firstMatchingFilter.set_local_pref ELSE receivedUpdate.local_pref END
WITH r2, received_route
MATCH (r2)-[:HAS_ROUTE]->(route:Route) WHERE route.prefix = received_route.prefix
SET route.best_path = false
WITH r2, route
ORDER BY route.local_pref DESC, route.med ASC, size(route.path) ASC
WITH r2, COLLECT(route) AS routes
SET HEAD(routes).best_path = true
WITH r2, id(HEAD(routes)) AS bestRouteId WHERE HEAD(routes).originRouteId = $route_id 
CALL custom.runBGP(bestRouteId) YIELD ignored
RETURN null AS ignored", 'write')
```

## Examples

Clean results before running an example:
```bash
MATCH (n) WHERE n.simulation IS NOT NULL
MATCH ()-[r]-() WHERE r.simulation IS NOT NULL
DELETE r
DETACH DELETE n
```

### Denied route

1. Simulate route propagation:
```bash
MATCH (r1:Router{name:'leaf01'})
MERGE (newRoute:Route {prefix: "10.0.0.0/8", origin_as: r1.asn, local_pref: 300, med: 0, next_hop: "192.0.2.2", best_path: true, communities: [], path: [], simulation: true})
MERGE (r1)-[r:HAS_ROUTE {simulation: true}]->(newRoute)
WITH newRoute
CALL custom.runBGP(id(newRoute)) YIELD ignored
RETURN null as ignored;
```

2. Print filters applied to the route:
```bash
MATCH (route:Route)<-[]-(filter:BgpFilter)<-[:HAS_BGP_FILTER]-(router:Router)
WHERE route.prefix = '10.0.0.0/8'
RETURN route, filter, router;
```


### Accepted route

1. Simulate route propagation:
```bash
MATCH (r1:Router{name:'leaf01'})
MERGE (newRoute:Route {prefix: "10.11.200.0/24", origin_as: r1.asn, local_pref: 300, med: 0, next_hop: "192.0.2.2", best_path: true, communities: [], path: [], simulation: true})
MERGE (r1)-[r:HAS_ROUTE {simulation: true}]->(newRoute)
WITH newRoute
CALL custom.runBGP(id(newRoute)) YIELD ignored
RETURN null as ignored;
```

2. Print installed routes:
```bash
MATCH (r:Router)-[:HAS_ROUTE]->(rr:Route)
WHERE rr.prefix STARTS WITH '10.11.200.0'
RETURN r,rr;
```

3. Print the complete propagation path of a route: 

```bash
MATCH path=(l02:Router{name:'leaf02'})-[:HAS_ROUTE]->(route:Route)-[:LEARNED_FROM]-(from_update:BgpUpdate)-[:RECEIVES_UPDATE]-(p:BgpPeer)-[:HAS_BGP_SESSION]-(sp:BgpPeer)<-[:HAS_BGP_PEER]-(:Port)<-[:HAS_PORT]-(from_router:Router)
WHERE route.prefix STARTS WITH '10.11.200.0'
RETURN path;
```