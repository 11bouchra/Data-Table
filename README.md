The core concept follows a similar approach to the "Find Usage" feature in IDEA, identifying the impact of code changes and iteratively traversing the affected classes and methods until the top-level controller layer is reached.

The code is primarily written in Python and utilizes two libraries:

javalang: A library used to parse Java file syntax.
unidiff: A library for parsing git diff information.
By using javalang for syntax parsing, the tool extracts information from each Java file, such as imports, classes, inheritance (extends), interfaces (implements), fields, methods, etc. The parsed data is stored in an SQLite3 database, divided into tables like project, class, import, field, and methods, each holding relevant information. SQL queries are then used to retrieve method-related data.

On the other hand, unidiff is employed to parse git diff files, tracking added and removed lines of code. Based on this information, the tool identifies which classes and methods are impacted, and continues to traverse these affected elements until it reaches the top-level controller layer.

SQLite3 Table Structure
The table structure is as follows:

SQL
Shrink ▲   
CREATE TABLE project (
	project_id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
	project_name TEXT NOT NULL,
    git_url TEXT NOT NULL,
    branch TEXT NOT NULL,
    commit_or_branch_new TEXT NOT NULL,
    commit_or_branch_old TEXT,
    create_at TIMESTAMP NOT NULL DEFAULT (datetime('now','localtime'))
);

CREATE TABLE class (
	class_id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
	filepath TEXT,
	access_modifier TEXT,
	class_type TEXT NOT NULL,
	class_name TEXT NOT NULL,
	package_name TEXT NOT NULL,
	extends_class TEXT,
	project_id INTEGER NOT NULL,
	implements TEXT,
	annotations TEXT,
	documentation TEXT,
	is_controller REAL,
	controller_base_url TEXT,
	commit_or_branch TEXT,
	create_at TIMESTAMP NOT NULL DEFAULT (datetime('now','localtime'))
);

CREATE TABLE import (
	import_id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
	class_id INTEGER NOT NULL,
	project_id INTEGER NOT NULL,
	start_line INTEGER,
	end_line INTEGER,
	import_path TEXT,
	is_static REAL,
	is_wildcard REAL,
    create_at TIMESTAMP NOT NULL DEFAULT (datetime('now','localtime'))
);

CREATE TABLE field (
	field_id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
	class_id INTEGER,
	project_id INTEGER NOT NULL,
	annotations TEXT,
	access_modifier TEXT,
	field_type TEXT,
	field_name TEXT,
	is_static REAL,
	start_line INTEGER,
	end_line INTEGER,
	documentation TEXT,
    create_at TIMESTAMP NOT NULL DEFAULT (datetime('now','localtime'))
);


CREATE TABLE methods (
	method_id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
	class_id INTEGER NOT NULL,
	project_id INTEGER NOT NULL,
	annotations TEXT,
	access_modifier TEXT,
	return_type TEXT,
	method_name TEXT NOT NULL,
	parameters TEXT,
	body TEXT,
	method_invocation_map TEXT,
	is_static REAL,
	is_abstract REAL,
	is_api REAL,
	api_path TEXT,
	start_line INTEGER NOT NULL,
	end_line INTEGER NOT NULL,
	documentation TEXT,
    create_at TIMESTAMP NOT NULL DEFAULT (datetime('now','localtime'))
    
);This section focuses on the method_invocation_map field in the methods table, which holds information about the classes and methods invoked by the parsed method. This allows for easy querying to identify which methods have used a specific class or method. Below is an example of the data stored in the method_invocation_map field:
Java
{
	"com.XXX.Account": {
		"entity": {
			"return_type": true
		}
	},
	"com.XXXX.AccountService": {
		"methods": {
			"functionAA(Map<String#Object>)": [303]
		},
		"fields": {
			"fieldBB": [132]
		}
	}
}
Analyzing Calls
This part of the main logic is changed to analyze which methods call the modified classes or methods, querying the results via SQL. Example code:

SQL
SELECT
	*
FROM
	methods
WHERE
	project_id = 1
	AND (json_extract(method_invocation_map,
	'$."com.xxxx.ProductUtil".methods."convertProductMap(QueryForCartResponseDTO)"') IS NOT NULL
	OR json_extract(method_invocation_map,
	'$."com.xxxx.ProductUtil".methods."convertProductMap(null)"') IS NOT NULL)
Display Method
Previously, the display method used tree diagrams and relational graphs. However, the tree diagrams did not clearly show the links, and the coordinates of the nodes in the relational graphs were not reasonable. This part has also been optimized. The node coordinates are calculated based on the node relationships. The longer the relationship link, the larger the horizontal coordinate, making the display clearer. Example code:

Java
Shrink ▲   
def max_relationship_length(relationships):
    if not relationships:
        return {}
    # Build adjacency list
    graph = {}
    for relationship in relationships:
        source = relationship['source']
        target = relationship['target']
        if source not in graph:
            graph[source] = []
        if target not in graph:
            graph[target] = []
        graph[source].append(target)

    # BFS Traverse and calculate the longest path length from each node to the starting point
    longest_paths = {node: 0 for node in graph.keys()}
    graph_keys = [node for node in graph.keys()]
    longest_paths[graph_keys[0]] = 0
    queue = deque([(graph_keys[0], 0)])
    while queue:
        node, path_length = queue.popleft()
        if not graph.get(node) and not queue and graph_keys.index(node) + 1 < len(graph_keys):
            next_node = graph_keys[graph_keys.index(node) + 1]
            next_node_path_length = longest_paths[next_node]
            queue.append((next_node, next_node_path_length))
            continue
        for neighbor in graph.get(node, []):
            if path_length + 1 > longest_paths[neighbor]:
                longest_paths[neighbor] = path_length + 1
                queue.append((neighbor, path_length + 1))
    return longest_paths
Display Effect
Image 1

Three Analysis Methods
JCCI can analyze three different scenarios: comparing two commits on the same branch, analyzing a specified class, and analyzing features across two branches. Example:

Java
from path.to.jcci.src.jcci.analyze import JCCI

# Comparison of different commits on the same branch
commit_analyze = JCCI('git@xxxx.git', 'username1')
commit_analyze.analyze_two_commit('master','commit_id1','commit_id2')

# Analyze the method impact of a class. The last parameter of the analyze_class_method method is the number of lines where the method is located. The number of lines of different methods is separated by commas. If left blank, the impact of the complete class will be analyzed.
class_analyze = JCCI('git@xxxx.git', 'username1')
class_analyze.analyze_class_method('master','commit_id1', 'package\src\main\java\ClassA.java', '20,81')

# Compare different branches
branch_analyze = JCCI('git@xxxx.git', 'username1')
branch_analyze.analyze_two_branch('branch_new','branch_old')
Flexible Configuration
You can configure the sqlite3 database storage path, project code storage path, and files to ignore during parsing in the config file. Example:

Java
db_path = os.path.dirname(os.path.abspath(__file__))
project_path = os.path.dirname(os.path.abspath(__file__))
ignore_file = ['*/pom.xml', '*/test/*', '*.sh', '*.md', '*/checkstyle.xml', '*.yml', '.git/*']
