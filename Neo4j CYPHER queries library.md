#results

___
- [[#🔧 Setup|🔧 Setup]]
- [[#🧠 Load data|🧠 Load data]]
	- [[#🧠 Load data#⏳Load graph from JSON file|⏳Load graph from JSON file]]
- [[#🧩 Node pre processing|🧩 Node pre processing]]
	- [[#🧩 Node pre processing#🏷️ Create different node types|🏷️ Create different node types]]
	- [[#🧩 Node pre processing#📑Add node attributes|📑Add node attributes]]
- [[#🔎 Explore the graph|🔎 Explore the graph]]
	- [[#🔎 Explore the graph#📈 Graph metrics|📈 Graph metrics]]
	- [[#🔎 Explore the graph#🔗 Graph operations|🔗 Graph operations]]
	- [[#🔎 Explore the graph#🛒 Filter nodes|🛒 Filter nodes]]
	- [[#🔎 Explore the graph#🧹 Filter edges|🧹 Filter edges]]
- [[#📂 Export data|📂 Export data]]
- [[#🚀 Advanced queries|🚀 Advanced queries]]
- [[#📊 Graph Data Science|📊 Graph Data Science]]
	- [[#📊 Graph Data Science#📦 Create graph object|📦 Create graph object]]
	- [[#📊 Graph Data Science#📈 Graph metrics|📈 Graph metrics]]
	- [[#📊 Graph Data Science#🔦 Community detection|🔦 Community detection]]
	- [[#🔦 Community detection#🕸️ Connected components|🕸️ Connected components]]
	- [[#📊 Graph Data Science#📈 Community metrics|📈 Community metrics]]
___

## 🔧 Setup

**Install required plugins**
- 📦 **APOC Plugin** - Install via Neo4j Desktop.
- 📦 **GDS Plugin** - Install via Neo4j Desktop.

**Configure the database**
- 📁 `apoc.conf` - Edit via Neo4j Desktop in `> folders > conf`. Create the `apoc.conf` file and add the following lines :
```
apoc.import.file.enabled=true
apoc.export.file.enabled=true
```
- ⚙️ **Settings** - Edit via Neo4j Desktop in `> Settings`. Un-comment and make sure the following option is parameterised as : 
```
dbms.memory.transaction.total.max=0
```

**Helpful links**
[APOC in Neo4j v5](https://towardsdatascience.com/what-happened-with-apoc-in-neo4j-v5-core-and-extended-edition-23994cdf0a2c)
[Importing large JSON files](https://github.com/neo4j/neo4j/issues/13008) 

📊 **Graph Data Science library** : [Getting Started with GDS](https://neo4j.com/docs/graph-data-science-client/current/getting-started/)
🧩 **Awesome Procedures On Cypher library** : [APOC user guide](https://neo4j.com/docs/apoc/current/()

____
## 🧠 Load data

### ⏳Load graph from JSON file

**Basic graph loading**
```
CALL apoc.load.json("KG_endometriosis.json") YIELD value
WITH value.nodes AS nodes, value.edges AS edges
FOREACH(n IN nodes |
	MERGE(p:Nodes{id: n.id})
	ON CREATE
	SET p.name = n.name,
		p.type = n.type,
		p.broaderDescriptor = n.broaderDescriptor,
		p.inchikey = n.inchikey,
		p.meshCategory = n.category,
        p.graph = "graph"
)
WITH edges
UNWIND edges AS edge
MATCH(p:Nodes{id: edge.source})
MATCH(m:Nodes{id: edge.target})
CALL apoc.create.relationship(p, edge.label, {}, m) YIELD rel
RETURN rel;
```

**Graph loading with multiple node types**
```
CALL apoc.load.json("KG_endometriosis.json") YIELD value
WITH value.nodes AS nodes, value.edges AS edges
FOREACH(n IN nodes |
	MERGE(p:Nodes{id: n.id})
	ON CREATE
	SET p.name = n.name,
		p.type1 = n.type1,
		p.type2 = n.type2,
		p.broaderDescriptor = n.broaderDescriptor,
		p.inchikey = n.inchikey
)
WITH edges
UNWIND edges AS edge
MATCH(p:Nodes{id: edge.source})
MATCH(m:Nodes{id: edge.target})
CALL apoc.create.relationship(p, edge.label, {}, m) YIELD rel
RETURN rel;
```

## 🧩 Node pre processing

### 🏷️ Create different node types

**Metabolites (chemical compounds)**
```
MATCH (n:Nodes {type: "metabolite"})
REMOVE n:Nodes
SET n:Nodes:Metabolites
```

**Biomedical concepts**
```
MATCH (n:Nodes {type: "concept"})
REMOVE n:Nodes
SET n:Nodes:Concepts
```

### 📑Add node attributes

**Set attribute to nodes directly related to endometriosis**
```
MATCH (c:Concepts {name:'Endometriosis'})<-[:`skos:related`]-(n:Nodes)
SET n.category = 'primary'
```

**Create new node type for chemical classes**
```
MATCH (m:Metabolites {inchikey: "null"})
REMOVE m:Metabolites
SET m:Metabolites:ChemicalClass
```

## 🔎 Explore the graph

**See everything**
```
MATCH (n)-[r]->(m) RETURN n, r, m;
```

**Count the different types of nodes**
Biomedical concepts
```
MATCH (n:Concepts)
RETURN count(n) as count
```
Metabolites
```
MATCH (n:Metabolites)
RETURN count(n) as count
```

**Count all nodes**
```
MATCH (n)
RETURN count(n) as count
```

**Count orphan nodes**
```
MATCH (n)
WHERE NOT EXISTS ((n)--())
RETURN COUNT(n)
```

**Count all edges**
```
MATCH ()-[r]->()
RETURN count(r) as count
```

**Delete everything**
```
MATCH (n)
DETACH DELETE n
```

### 📈 Graph metrics

**Average path length** 
```
MATCH (n:Nodes)
WITH collect(n) as nodes
UNWIND nodes as i
UNWIND nodes as j
WITH i, j WHERE id(i) < id(j)
MATCH path = shortestPath((i)-[*]-(j))
RETURN AVG(length(path))
```

**Diameter**
```
MATCH (n)
WITH collect(n) AS nodes
UNWIND nodes AS i
UNWIND nodes AS j
WITH i, j WHERE id(i) < id(j)
MATCH path=shortestPath((i)-[*]-(j))
RETURN length(path) AS diameter
ORDER BY diameter
DESC LIMIT 1
```

**Degree distribution**
```
MATCH (n)
RETURN size([(n)--() | 1]) AS degree, COUNT(n.id) AS frequency
ORDER BY degree ASC
```
### 🔗 Graph operations

**Add MeSH thesaurus hierarchy edges** (with tree numbers)
```
MATCH (n:Concepts)
MATCH (m:Concepts)
WHERE n.meSHParentTreeNb = m.meSHTreeNb
MERGE (n)-[:subTermOf]->(m)
RETURN *
```

**Match identical compounds** (owl:sameAs relationship)
```
MATCH (start)-[r:`owl:sameAs`]->(end)
RETURN start, r, end
```

### 🛒 Filter nodes

**Remove compounds that are linked to only one MeSH concept**
```
MATCH (n:Metabolites)
WHERE NOT EXISTS {
	MATCH (n)-[]-(c:Concepts)
	WITH n, COUNT(DISTINCT c) AS num_related_nodes
	WHERE num_related_nodes >= 2
	RETURN n
}
DETACH DELETE n
```

**Remove duplicated compounds**
duplicated compounds between PubChem and ChEBI -> keep the one from PubChem because the database is more extensive and it will be easier to gather additional information from the PMID rather than the ChEBI ID
```
MATCH (start)-[r:`owl:sameAs`]->(end)
DETACH DELETE end
```

**Remove endometriosis (concept) node**
```
MATCH (n:Concepts {name: "Endometriosis"})
DETACH DELETE n
```

**Remove higher concepts**
```
MATCH (c:Concepts)
WHERE (c)<-[:`meshv:broaderDescriptor`]-(:Concepts)
DETACH DELETE c
```

**Remove MeSH related to no compounds**
```
MATCH (c:Concepts)
WHERE NOT (c)<-[:`skos:related`]-(:Metabolites)
DETACH DELETE c
```

**Filter nodes by degree** 
```
MATCH (n:Concepts)<-[:`skos:related`]-(m:Metabolites)
WHERE n.degree > 90 AND m.degree > 80
RETURN n, m
```

### 🧹 Filter edges

**Delete duplicated relationships**
```
MATCH (m:Metabolites)-[x:`skos:related`]->(c:Concepts)
MATCH (m)-[y:`rdfs:subClassOf`]->(cl:ChemicalClasses)-[z:`skos:related`]->(c)
DELETE z
```

## 📂 Export data

**Export to JSON** (with filter)
```
CALL apoc.export.json.query(
"MATCH (n:Concepts)<-[r:related]-(m:Metabolites) WHERE n.degree > $degreeN AND m.degree > $degreeM RETURN COLLECT(n {.*}) AS concepts, COLLECT(m {.*}) AS compounds, COLLECT({source:m.id, target:n.id}) AS edges",
"endo_subgraph.json",
{params:{degreeN:100, degreeM:100}}
)
```

**Export to JSON** (all nodes)
```
CALL apoc.export.json.query(
"MATCH (n:Concepts)<-[r:related]-(m:Metabolites) RETURN COLLECT(n {.*}) AS concepts, COLLECT(m {.*}) AS compounds, COLLECT({source:m.id, target:n.id}) AS edges",
"endo_subgraph.json"
)
```

## 🚀 Advanced queries

**Match all nodes with the same value for an attribute**
-> match metabolites with the same InChiKey (the graph is built nicely so compounds can't have the same InChiKey, but it matches the ones with a `null` InChiKey. At least this is an example query on how to match stuff with CYPHER 🫠)
```
MATCH (n:Metabolites)
WITH n.inchikey AS inchikey, COLLECT(n) AS nodes
WHERE SIZE(nodes) > 1
RETURN inchikey, nodes
```

**Nodes collapse**
only collapses compounds into their chemical classes, keeps the chemical classes hierarchy
```
MATCH (m:Metabolites)-[:`rdfs:subClassOf`]->(c:ChemicalClasses)
WITH m, collect(c) AS subgraph
CALL apoc.nodes.collapse(subgraph, {
	properties: 'combine'
})
YIELD from, rel, to
RETURN from, rel, to
```

**Merge nodes and remove relationship**
```
MATCH (n:Metabolites)-[:`owl:sameAs`]->(p:Metabolites)
CALL apoc.refactor.mergeNodes([n, p], {properties: "combine", mergeRels: true})
YIELD node
MATCH (:Metabolites)-[r:`owl:sameAs`]->(:Metabolites)
DELETE r
RETURN node
```

**Find the sub graph associated to experimental data**
```
MATCH (data:Metabolites)-[:`experimental data`]->(m:Metabolites)-[]->(c:Concepts)
RETURN data, m, c
```

**Get the shared concepts**
```
MATCH (data1:Metabolites)-[:`experimental data`]->(m1:Metabolites)-[]->(c:Concepts)<-[]-(m2:Metabolites)-[:`experimental data`]-(data2:Metabolites)
RETURN data1, data2, m1, m2, c
```

**Graph grouping**
```
CALL apoc.nodes.group(['ChemicalClasses', 'Concepts', 'Metabolites'],['nodeType'])
YIELD nodes, relationships
RETURN *
```

## 📊 Graph Data Science

<u>modes</u> :
- **stream** : returns the result of the algorithm as a stream of records
- **stats** : returns a single record of summary statistics
- **mutate** : writes the results of the algorithm to the projected graph and returns a single record of summary statistics
- **write** : writes the results of the algorithm to the Neo4j database and returns a single record of summary statistics (❗ don't use that unless you are very very <u>very</u> certain that you want to have the results of the algorithm in the Neo4j database)

## 📦 Graph projection

❗ the graph projection has to be done every time you re open the Neo4j browser, it does not save it 
**Project the graph**
```
CALL gds.graph.project(
	'endometriosis',
	['Metabolites', 'Concepts', 'ChemicalClasses'],
	['skos:related', 'experimental data','rdfs:subClassOf','owl:sameAs','meshv:broaderDescriptor']
)
YIELD graphName AS graph, nodeProjection, nodeCount AS nodes, relationshipProjection, relationshipCount AS rels
```
-> CYPHER projection : creates an in-memory graph from a CYPHER query
-> Project a graph a each step of the graph pruning / sparsification to list the metrics easily with `gds.graph.list()`

**Force an undirected graph projection**
for each relation, specify that you want to project it undirected
```
CALL gds.graph.project(
	'endometriosisUndirected',
	['Metabolites', 'Concepts', 'ChemicalClasses'],
	{
		`skos:related`: {
			type: 'skos:related',
			orientation: 'UNDIRECTED'
		},
		`experimental data`: {
			type: 'experimental data',
			orientation: 'UNDIRECTED'
		},
		`rdfs:subClassOf`: {
			type: 'rdfs:subClassOf',
			orientation: 'UNDIRECTED'
		}
	}
)
YIELD graphName AS graph, nodeProjection, nodeCount AS nodes, relationshipProjection, relationshipCount AS rels
```

### 📈 Graph metrics

**Node and edge counts**
```
CALL gds.graph.list('endometriosis')
YIELD graphName, nodeCount, relationshipCount
RETURN graphName, nodeCount, relationshipCount
ORDER BY graphName ASC
```

**Specify an undirected relationship**
```
CALL gds.graph.relationships.toUndirected(
	'endometriosis',
	{relationshipType: 'skos:related', mutateRelationshipType: 'skos:related_undir'}
)
YIELD inputRelationships, relationshipsWritten
```

**Density**
```
CALL gds.graph.list('endometriosis')
YIELD graphName, density
RETURN graphName, density
ORDER BY graphName ASC
```

**Degree distribution**
```
CALL gds.graph.list('endometriosis')
YIELD graphName, degreeDistribution
RETURN graphName, degreeDistribution
ORDER BY graphName ASC
```

**Find hubs in the network (core structure)**
example with betweenness centrality (`gds.betweenness.stream`)
```
CALL gds.betweenness.stream('endometriosis')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC
```
other centrality algorithms in the GDS library : `gds.articleRank` `gds.articulationPoints` `gds.influenceMaximization.celf` `gds.closeness` `gds.degree` `gds.eigenvector` `gds.pageRank` `gds.closeness.harmonic` `gds.hits` 

**Shortest path**
example with Dijkstra Source-Target Shortest Path
```
MATCH (source:Metabolites {name: '3-oxo-Delta(4) steroid'}), (target:Metabolites {name: '20-oxo steroid'})
CALL gds.shortestPath.dijkstra.stream('myGraph', {
    sourceNode: source,
    targetNodes: target,
    relationshipWeightProperty: 'cost'
})
YIELD index, sourceNode, targetNode, totalCost, nodeIds, costs, path
RETURN
    index,
    gds.util.asNode(sourceNode).name AS sourceNodeName,
    gds.util.asNode(targetNode).name AS targetNodeName,
    totalCost,
    [nodeId IN nodeIds | gds.util.asNode(nodeId).name] AS nodeNames,
    costs,
	nodes(path) as path 
ORDER BY index
```
with several targets :
```
MATCH (source:Metabolites {name: '3-oxo-Delta(4) steroid'}), (targetA:Metabolites {name: '20-oxo steroid'}), (targetB:Metabolites {name: 'Progesterone'})
CALL gds.shortestPath.dijkstra.stream('endometriosis', {
	sourceNode: source,
	targetNodes: [targetA, targetB]
})
YIELD ...
```

**Bridges**
❗ undirected graph 
```
CALL gds.bridges.stream('endometriosisUndirected')
YIELD from, to
RETURN gds.util.asNode(from).name AS fromName, gds.util.asNode(to).name AS toName
ORDER BY fromName ASC, toName ASC
```

### 🔦 Community detection

**K-core decomposition**
❗ undirected graph 
stream
```
CALL gds.kcore.stream('endometriosisUndirected')
YIELD nodeId, coreValue
RETURN gds.util.asNode(nodeId).name AS name, coreValue
ORDER BY coreValue ASC, name DESC
```
stats
```
CALL gds.kcore.stats('endometriosisUndirected')
YIELD degeneracy
RETURN degeneracy
```
R friendly
```
CALL gds.kcore.stream('endometriosisUndirected')
YIELD nodeId, coreValue
RETURN gds.util.asNode(nodeId).id AS id, coreValue AS community
ORDER BY community ASC, id DESC
```

**K-1 colouring**
stream
```
CALL gds.k1coloring.stream('endometriosis')
YIELD nodeId, color
RETURN gds.util.asNode(nodeId).name AS name, color
ORDER BY name
```
stats
```
CALL gds.k1coloring.stats('endometriosis')
YIELD nodeCount, colorCount, ranIterations, didConverge
```
mutate (same for all other community detection methods)
```
CALL gds.k1coloring.mutate('endometriosis', {
	mutateProperty: 'color'
})
YIELD nodeCount, colorCount, ranIterations, didConverge
```
R friendly
```
CALL gds.k1coloring.stream('endometriosis')
YIELD nodeId, color
RETURN gds.util.asNode(nodeId).id AS id, color AS community
ORDER BY community ASC, id DESC
```

**K-means clustering**
❗ needs a numeric property 
❗ elbow curve method to estimate the best value of k

**Label propagation**
stream
```
CALL gds.labelPropagation.stream('endometriosis')
YIELD nodeId, communityId AS Community
RETURN gds.util.asNode(nodeId).name AS Name, Community
ORDER BY Community, Name
```
stats
```
CALL gds.labelPropagation.stats('endometriosis')
YIELD communityCount, ranIterations, didConverge
```
size of communities (same for all other community detection methods)
```
CALL gds.labelPropagation.stream('endometriosis')
YIELD nodeId, communityId
RETURN communityId, COUNT(nodeId) AS communitySize
ORDER BY communitySize DESC
```

**Leiden**
❗ undirected graph 
stream
```
CALL gds.leiden.stream('endometriosisUndirected', { 
	randomSeed: 19 
})
YIELD nodeId, communityId
RETURN gds.util.asNode(nodeId).name AS name, communityId
ORDER BY communityId ASC
```
stats
```
CALL gds.leiden.stats('endometriosisUndirected', { randomSeed: 19 })
YIELD communityCount
```

**Louvain**
stream
```
CALL gds.louvain.stream('endometriosis')
YIELD nodeId, communityId, intermediateCommunityIds
RETURN gds.util.asNode(nodeId).name AS name, communityId
ORDER BY communityId ASC
```
stats
```
CALL gds.louvain.stats('endometriosis')
YIELD communityCount
```

**Approximate maximum k-cut**
stream
```
CALL gds.maxkcut.stream('endometriosis')
YIELD nodeId, communityId
RETURN gds.util.asNode(nodeId).name AS name, communityId
```

**Speaker-listener label propagation**
stream
```
CALL gds.sllpa.stream('endometriosis', {
	maxIterations: 100, 
	minAssociationStrength: 0.1
})
YIELD nodeId, values
RETURN gds.util.asNode(nodeId).name AS Name, values.communityIds AS communityIds
ORDER BY Name ASC
```

### 🕸️ Connected components

**Strongly connected components**
```
CALL gds.scc.stream('endometriosis', {})
YIELD nodeId, componentId
RETURN gds.util.asNode(nodeId).name AS Name, componentId AS Component
ORDER BY Component, Name DESC
```

**Weakly connected components**
```
CALL gds.wcc.stream('endometriosis')
YIELD nodeId, componentId
RETURN gds.util.asNode(nodeId).name AS name, componentId
ORDER BY componentId, name
```

**Number of connected components**
strongly connected components
```
CALL gds.scc.stats('endometriosis')
YIELD componentCount
```
weakly connected components
```
CALL gds.wcc.stats('endometriosis')
YIELD componentCount
```
(only one component with wcc on my graph -> look into it #todo)

**Size of connected components**
```
CALL gds.scc.stream('endometriosis')
YIELD nodeId, componentId
RETURN componentId, COUNT(nodeId) AS componentSize
ORDER BY componentSize DESC
```

### 📈 Community metrics

❗ all examples with the `color` communities from k1-colouring

**Conductance metric**
```
CALL gds.conductance.stream('endometriosis', {
	communityProperty: 'color' 
})
YIELD community, conductance
```

**Modularity**
stream
```
CALL gds.modularity.stream('endometriosis', {
	communityProperty: 'color'
})
YIELD communityId, modularity
RETURN communityId, modularity
ORDER BY communityId ASC
```
stats
```
CALL gds.modularity.stats('endometriosis', {
	communityProperty: 'color'
})
YIELD nodeCount, relationshipCount, communityCount, modularity
```

**Clustering coefficient**
❗ undirected graph 
```
CALL gds.localClusteringCoefficient.stream('endometriosisUndirected')
YIELD nodeId, localClusteringCoefficient
RETURN gds.util.asNode(nodeId).name AS name, localClusteringCoefficient
ORDER BY localClusteringCoefficient DESC, name
```
stats
```
CALL gds.localClusteringCoefficient.stats('endometriosisUndirected')
YIELD averageClusteringCoefficient, nodeCount
```

**Triangle count**
❗ undirected graph 
stream
```
CALL gds.triangleCount.stream('endometriosisUndirected')
YIELD nodeId, triangleCount
RETURN gds.util.asNode(nodeId).name AS name, triangleCount
ORDER BY triangleCount DESC, name ASC
```
stats
```
CALL gds.triangleCount.stats('endometriosisUndirected')
YIELD globalTriangleCount, nodeCount
```
