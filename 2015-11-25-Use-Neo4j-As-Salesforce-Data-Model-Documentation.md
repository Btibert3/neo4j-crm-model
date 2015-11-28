# Use Neo4j to Document Salesforce Requirements

I have been working on a team that is aiming to implement a Salesforce-based CRM solution for Enrollment Management.  From the beginning, we had an agressive timeline, and the project has taken many twists and turns along the way. While our experience is certainly not unique, and perhaps commonplace, it's provided us with an opportunity to evaluate some of the fundamental steps that should be set in place prior to continuing down our deployment path considering our go-live date is currently TBD.    

Namely:

- Solidifying our data model.  To date, we still don't have this set, and only focused on "small wins" inside an incomplete data model.
- Doing the due-diligence to identify business processes and implement these via Salesforce functionality like `workflow`, `validation rules`, and `Process Builder`.  Why worry about the electricity in the bathroom if you haven't poured the concrete for the entire house?  The key thing here is to implement the business process __within__ Salesforce, not through data integration procedures.
- Document what is under the hood; simply, the elements that drive our application's logic.  Documentation always feels like an afterthought, so why not make it a first-class citizen since we are already starting with a clean slate?

## Why do this

Recently I was working to prototype a business need to read and evaluate applications within Salesforce.  From the beginning, I have always viewed an object's `page layout` to be what my team today refers to as our "E-form", or the screen that we use to read an application online.  

While working through my problem, it occurred to me that having a firm grasp of ___what depends on what___ would be crucial to documenting our business, as you are building out the rules that our Admissions folks will be relying on.  

Even though its core function is to serve as a graph database, [Neo4j](http://neo4j.com/) and it's [cypher](http://neo4j.com/developer/cypher-query-language/) langugage can act as an excellent tool to solve the documentation problem.  

`Nodes` can be things like:
- Objects,
- fields on those objects,
- properties of those fields, or better yet,
- processes driven through `workflow`.  

`Relationships` between `Nodes` can:  
- Identify what fields are on what object  
- Document how objects are related to each other, and record the type of relationship  
- `Picklist` values can be mapped  
- In the case of `workflow` rules, you can document which object the rule is applied to, what fields are being evaluated, and most importantly in the case of a field update, what field the rule updates.  

## The Data Model

This post will implement a very basic, high-level CRM in Salesforce that consists of just two objects:

1.  The standard object `Contact`  
2.  A custom object `Application`

By no means am I rolling out a robust proof-of-concept, but the aim is to highlight how we can leverage `Neo4j` to visualize what is going on within our Salesforce Org.  With a few simple queries, we can return visizualizations that make it easier to understand what is actually going on under the hood, something that is invaluable when trying to quickly resolve issues.

The __Appendix__ at the end of this post includes the code that I used to generate the data model.  Because `cypher` is really easy to read, developing your model is pretty straightforward.  

## Explore

Enough with the setup.  Let's start to take a look at the data model that I have setup for this exercise.  

```cypher
// my data model
MATCH (a)-[r]->(b)
WHERE labels(a) <> [] AND labels(b) <> []
RETURN DISTINCT head(labels(a)) AS This, type(r) as To, head(labels(b)) AS That
```

returns

![data-model-table](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/data-model-table.PNG)

We can see what is related to each other, and how. For example, Object's have fields, but a field can be related to other objects and have a type (i.e. Lookup relationship).

And let's put a picture to that model.

![data-model-graph](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/data-model-graph.PNG)

But that is just the foundation.  Let's start to ask questions of the model to highlight how this can help us in the long run.

#### What objects are in our org?

```cypher
MATCH (n) RETURN n;
```
![objectsh](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/objects.PNG)

#### Show Everythign related to the Contact Object

```cypher
// Everything related to the contact object
MATCH (app:Object {name:'Contact'})-[*]->(n) RETURN app, n
```
![objectsh](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/contact-object.PNG)

From the graph, we can see our Contact object has 4 fields, and while not labeled, Gender is a `picklist` with two values, Male and Female.

#### Show Everything related to the Application Object

```cypher
// Everything related to the app object
MATCH (app:Object {name:'Application'})-[*]->(n) RETURN app, n
```
![app-depends](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/app-object.PNG)

This is a little more complicated, and for the purposes of a high-level view, it's not easy to see everything going on.  We will explore this later, but the key is that we identifed that the Application object is related to the App object via a field on `Contact` on the application.


#### What fields are picklists on the Application

```cypher
// what fields does the app object that are picklists
MATCH (n:Object {name:'Application'})-[:HAS_FIELD]->(f:Field {type:'Picklist'})
RETURN n, f
```

![app-pls](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/app-picklists.PNG)

We will come back to this, but above I have two fields, First and Second Read Decision.  In this example, I am indicating that a particular admissions office might have a business need to read the application twice.  This model is capturing each reader's thoughts independently.

#### How do we represent our admission  Funnel?

```cypher
// the values for the Funnel Status
MATCH (f:Field {name:'Funnel Status'})-[:HAS_VALUE]->(v)
RETURN f,v
```

![funnel](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/funnel.PNG)

This is particuarly important.  We are:

1.  Capturing the various states of the funnel that we commonly refer to.  
2.  We are also putting order to each state by using a `:NEXT` relationship.  The whole point of the funnel is that we can push our suspects or inquiries through the funnel, and eventually there end up in 1 of 3 states; registered, canceled, or denied.


#### Let's show the funnel in action

```cypher
// the steps of the funnel status
MATCH (app:Object {name:'Application'})-[:HAS_FIELD]->(f:Field {name:'Funnel Status'}),
      (f)-[:HAS_VALUE]->(v)
WITH f,v
MATCH (v)-[:NEXT]->(z)
RETURN v,z
```

By using `Neo4j` in this way, we are documenting how our business thinks about leads and how they need to progress.

![funnel](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/funnel-flow.PNG)

By following the paths, it's pretty easy to visualize how our Enrollment Management shop views the funnel, and how applicants will proceed through the admissions process.

> We would use these paths to enforce and document the rules within Salesforce to ensure our pool follows this process.

#### Validation Rule

Now let's get to the meat of the idea.  The code below identifies

```cypher
// look at a validation rule by business process ID as graph
MATCH (v:ValidationRule)-[r]->(n)
WHERE v.brID = 2.15
RETURN v,r,n
```

![validrule](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/validrule.PNG)

In this post, I only am building out one validation rule, but the aim is to higlighlt how this could be any business logic (or process) that your org needs.

From above, we can easily see that:

1.  The validation rule is applied to the Application Object  
2.  Which fields the validation rule is using  
3.  Which field I am validating.  In this case, the graph is showing that I am validating that when the checkbox First Read Complete is checked, I ensure that First Read Decision and First Read Comments are proprely filled out.  

But most importantly, I am querying the validation rule by it's documented `Business Requirement` id, or `brID=2.15`.  Needless to say, when gather requirements, more than likely you are capturing the business needs in a spreadsheet, and giving each an id for quick reference.  In this example, we are storing that id within `Neo4j` to enable a quick visual reference as to the dependencies of that requirement.

If you wanted to, it's possible to display this validation rule as a ruleset.  

```cypher
// look at a validation rule by business process ID as ruleset
MATCH (v:ValidationRule)-[r]->(n)
WHERE v.brID = 2.15
RETURN DISTINCT head(labels(v)) AS This, type(r) as To, n.name AS That
```

![ruleset](https://dl.dropboxusercontent.com/u/15276022/blog-images/neo4j-salesforce-documentation/ruleset.PNG)


## Summary

Hopefully this got you thinking about various ways you can leverage other tools to help your business implement a new Salesforce org. In my case, I am a massive fan of `Neo4j`, and it just feels natural to me leverage it as a documentation tool for Salesforce, given how complex things can get as you translate your business needs and logic from one solution to Salesforce.  In my case, I am coming at it from Enrollment Management within higher ed, but I can't see why this approach wouldn't apply to nearly all use-cases.

If you haven't heard of, or played around with `Neo4j`, I highly recommend that you check it out.  Beyond being a highly flexible `NoSQL` solution, the browser - which I used for all of my screenshots - is an incredible tool.

Let me know what you think.  By no means am I a Salesforce expert, but the more I use the platform, the more I am blown away at how easy it is to encapsulate core business needs.


## Appendix

Below is the cypher that I used to develop the graph for this post.

```sh
//Create the nodes
// Generated most of this in google sheets
CREATE (n1: Object {name:'Contact'})
CREATE (n2: Field {name:'First Name', type:'Text'})
CREATE (n3: Field {name:'Last Name', type:'Text'})
CREATE (n4: Field {name:'Email', type:'Email'})
CREATE (n5: Field {name:'Gender', type:'Picklist'})
CREATE (n6: Value {name:'Female'})
CREATE (n7: Value {name:'Male'})
CREATE (n8: Object {name:'Application'})
CREATE (n9: Field {name:'Term', type:'Picklist'})
CREATE (n10: Value {name:'Fall 2016'})
CREATE (n11: Value {name:'Fall 2017'})
CREATE (n12: Field {name:'Contact', type:'Lookup'})
CREATE (n13: Field {name:'Student Type', type:'Picklist'})
CREATE (n14: Value {name:'Freshmen'})
CREATE (n15: Value {name:'Transfer'})
CREATE (n16: Field {name:'Funnel Status', type:'Picklist'})
CREATE (n17: Value {name:'Suspect'})
CREATE (n18: Value {name:'Inquiry'})
CREATE (n19: Value {name:'App Started'})
CREATE (n20: Value {name:'App Submitted'})
CREATE (n21: Value {name:'App Complete'})
CREATE (n22: Value {name:'Admitted'})
CREATE (n23: Value {name:'Denied'})
CREATE (n24: Value {name:'Waitlisted'})
CREATE (n25: Value {name:'Deposited'})
CREATE (n26: Value {name:'Registered'})
CREATE (n27: Value {name:'Canceled'})
CREATE (n28: Field {name:'First Read Comments', type:'Text'})
CREATE (n29: Field {name:'Second Read Comments', type:'Text'})
CREATE (n30: Field {name:'First Read Decision', type:'Picklist'})
CREATE (n31: Field {name:'Second Read Decision', type:'Picklist'})
CREATE (n32: Value {name:'Accept'})
CREATE (n33: Value {name:'Deny'})
CREATE (n34: Value {name:'Defer'})
CREATE (n35: Value  {name:'Waitlist'})
CREATE (n36: Field {name:'First Read Complete', type:'Checkbox'})
CREATE (n37: Field {name:'Second Read Complete', type:'Checkbox'})
CREATE (n38: ValidationRule {name:'First Read Validation', brID:2.15})


// Create the relationships
CREATE (n1)-[:HAS_FIELD]->(n2)
CREATE (n1)-[:HAS_FIELD]->(n3)
CREATE (n1)-[:HAS_FIELD]->(n4)
CREATE (n1)-[:HAS_FIELD]->(n5)
CREATE (n5)-[:HAS_VALUE]->(n6)
CREATE (n5)-[:HAS_VALUE]->(n7)
CREATE (n8)-[:HAS_FIELD]->(n9)
CREATE (n9)-[:HAS_VALUE]->(n10)
CREATE (n9)-[:HAS_VALUE]->(n11)
CREATE (n8)-[:HAS_FIELD]->(n12)
CREATE (n12)-[:RELATED_TO]->(n1)
CREATE (n8)-[:HAS_FIELD]->(n13)
CREATE (n13)-[:HAS_VALUE]->(n14)
CREATE (n13)-[:HAS_VALUE]->(n15)
CREATE (n8)-[:HAS_FIELD]->(n16)
CREATE (n16)-[:HAS_VALUE]->(n17)
CREATE (n16)-[:HAS_VALUE]->(n18)
CREATE (n16)-[:HAS_VALUE]->(n19)
CREATE (n16)-[:HAS_VALUE]->(n20)
CREATE (n16)-[:HAS_VALUE]->(n21)
CREATE (n16)-[:HAS_VALUE]->(n22)
CREATE (n16)-[:HAS_VALUE]->(n23)
CREATE (n16)-[:HAS_VALUE]->(n24)
CREATE (n16)-[:HAS_VALUE]->(n25)
CREATE (n16)-[:HAS_VALUE]->(n26)
CREATE (n16)-[:HAS_VALUE]->(n27)
CREATE (n17)-[:NEXT]->(n18)
CREATE (n18)-[:NEXT]->(n19)
CREATE (n19)-[:NEXT]->(n20)
CREATE (n20)-[:NEXT]->(n21)
CREATE (n21)-[:NEXT]->(n22)
CREATE (n21)-[:NEXT]->(n23)
CREATE (n21)-[:NEXT]->(n24)
CREATE (n22)-[:NEXT]->(n25)
CREATE (n22)-[:NEXT]->(n27)
CREATE (n24)-[:NEXT]->(n22)
CREATE (n24)-[:NEXT]->(n23)
CREATE (n25)-[:NEXT]->(n26)
CREATE (n26)-[:NEXT]->(n27)
CREATE (n8)-[:HAS_FIELD]->(n28)
CREATE (n8)-[:HAS_FIELD]->(n29)
CREATE (n8)-[:HAS_FIELD]->(n30)
CREATE (n8)-[:HAS_FIELD]->(n31)
CREATE (n30)-[:HAS_VALUE]->(n32)
CREATE (n30)-[:HAS_VALUE]->(n33)
CREATE (n30)-[:HAS_VALUE]->(n34)
CREATE (n30)-[:HAS_VALUE]->(n35)
CREATE (n31)-[:HAS_VALUE]->(n32)
CREATE (n31)-[:HAS_VALUE]->(n33)
CREATE (n31)-[:HAS_VALUE]->(n34)
CREATE (n31)-[:HAS_VALUE]->(n35)
CREATE (n8)-[:HAS_FIELD]->(n36)
CREATE (n8)-[:HAS_FIELD]->(n37)

// Apply the validation rule
CREATE (n38)-[:APPLIED_TO]->(n8)
CREATE (n38)-[:EVALUATES_FIELD]->(n28)
CREATE (n38)-[:EVALUATES_FIELD]->(n30)
CREATE (n38)-[:VALIDATES]->(n36);

```
