# Semantic Whitepaper Code Example
This project provides the code example for the Whitepaper *Semantic IoT Solutions - A Developer Perspective* under [DOI 10.13140/RG.2.2.16339.53286](http://dx.doi.org/10.13140/RG.2.2.16339.53286).

In this section we provide an example written in Java on how to use the semantic information from an application. This does not mean that Java is the only or best language to use for programming semantic systems; Python could have been used just as well or another programming language of your choice.

As explained in the whitepaper, the semantic information needs to be stored in a triple store and we will use the SPARQL language to retrieve the information. So the first thing is to decide on which triple store to use as explained in section “Storing Semantic Information” of the whitepaper. In our example, we will use [Apache Jena Fuseki](https://jena.apache.org/documentation/fuseki2/) that provides a robust, transactional persistent storage layer and a SPARQL server.

The power profile instantiation example uses the S4ENER extension of SAREF ontology. Firstly, these models are stored within the triple store and the dataset is named (e.g., the dataset name is `energyds` and the dataset URI is `http://localhost:3030/energyds`). `localhost` is used here for our small example - for any real system a dereferenceable and reachable URI should be used.

To add a model to the dataset, the path file or URL and the RDF I/O technology (RIOT) are provided, e.g. Turtle, RDF/XML, etc. as shown below. The loadEnergyExample method shows how to add SAREF4ENER, SAREF and the power profile instantiation to the dataset. The file [example.ttl](https://raw.githubusercontent.com/martin-p-bauer/SemanticWhitepaperCodeExample/master/example.ttl) contains the RDF code for the power profile. SAREF4ENER is in Turtle format whereas SAREF is in RDF/XML format. 

```java
import org.apache.jena.query.*;
import org.apache.jena.rdf.model.*;
import java.io.*;
import java.util.*;
import org.apache.jena.update.*;
 
public void addFileModel(File rdf, String riot) throws IOException {
    // parse the ontology file
    Model newModel = ModelFactory.createDefaultModel();
    try (FileInputStream in = new FileInputStream(rdf)) {
        newModel.read(in, null, riot);
    }
    DatasetAccessor accessor = DatasetAccessorFactory.createHTTP(“http://localhost:3030/energyds”);
    // Add statements to the default model of a Dataset
    accessor.add(newModel);      
}
 
public void addUrlModel (String urlString, String riot) throws Exception {
     // parse the ontology file
     Model newModel = ModelFactory.createDefaultModel(); 
     // create the url
URL url = new URL(urlString);
     newModel.read(url.openStream(), null, riot);
     DatasetAccessor accessor = DatasetAccessorFactory.createHTTP(“http://localhost:3030/energyds”);
     // Add statements to the default model of a Dataset
     accessor.add(newModel);       
}
 
public void loadEnergyExample () {
    String sarefOntology ="https://saref.etsi.org/saref";
    String saref4EnerOntology ="https://saref.etsi.org/saref4ener";
    String example ="example.ttl";
    try {
        addUrlModel(sarefOntology, "RDF/XML");
        addUrlModel(saref4EnerOntology, "TURTLE");
        addFileModel(new File(example), "TURTLE");
    } catch (Exception ex) {
        System.err.println (ex.getMessage());
    }
}
```

Once the dataset is stored in the triple store, the application can query the dataset and ontology with the SPARQL language. To retrieve the values of the measurements of the heating system from the energy use case, the method getMeasurements is defined, which prints the properties, values and units of the measurements from the `PowerProfile-1-HS0001` instance. The method `queryModel` queries the dataset by executing the SPARQL query (encoded as a string) provided as an input.


```java
public List<Map<String, Object>> queryModel(String sparqlQuery) {
    List<Map<String, Object>> mapResult = new ArrayList<>();
 
    try (QueryExecution qexec = QueryExecutionFactory.sparqlService("http://localhost:3030/energyds", sparqlQuery)) {
        org.apache.jena.query.ResultSet results;
        results = qexec.execSelect();
        while (results.hasNext()) {
            QuerySolution soln = results.nextSolution();
            mapResult.add(createMap(soln));
        }
    }
    return mapResult;
}
private Map<String, Object> createMap(QuerySolution querySolution) {
    Map<String, Object> result = new HashMap<>();
    Iterator<String> it = querySolution.varNames();
    while (it.hasNext()) {
        String varName = it.next();
        if (querySolution.get(varName) instanceof Literal) {
            result.put(varName, querySolution.getLiteral(varName));
        } else {
            result.put(varName, querySolution.getResource(varName));
        }
     }
     return result;
}
public void getMeasurements () {
    List<Map<String, Object>> qryResult;
    String sparqlQuery = "PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> "
                + "PREFIX s4ener: <https://saref.etsi.org/saref4ener##> "
                + "PREFIX saref: <https://saref.etsi.org/saref#> "
                + "SELECT ?x ?y ?m ?a ?b "
                + "WHERE {"
                + " ?x rdf:type s4ener:PowerProfile . "
                + " ?x s4ener:belongsTo s4ener:HeatingSystem . "
                + " ?x saref:isAbout ?y . "
                + " ?m saref:relatesToProperty ?y . "
                + " ?m saref:hasValue ?a . "
                + " ?m saref:isMeasuredIn ?b ."
                + " } ";
    qryResult = queryModel(sparqlQuery);
    int i = 0;
    while (i < qryResult.size()) {
        Resource resMeasurement = (Resource) qryResult.get(i).get("x"); 
        System.out.println (resMeasurement.getURI());
        System.out.println ("Property " + ((Resource)qryResult.get(i).get("y")).getLocalName());
        System.out.println ("Value " + qryResult.get(i).get("a"));
        System.out.println ("Unit " + ((Resource)qryResult.get(i).get("b")).getLocalName());
        i++;
    }
}
```

The output of invoking the method `getMeasurements` with the instances in the file [example.ttl](https://raw.githubusercontent.com/martin-p-bauer/SemanticWhitepaperCodeExample/master/example.ttl) are shown below.

```
s4ener:PowerProfile-1-HS0001
Property Power_1
Value 0.2
Unit kilowatt
s4ener:PowerProfile-1-HS0001
Property Energy_1
Value 0.2
Unit kilowatt_hour
```
