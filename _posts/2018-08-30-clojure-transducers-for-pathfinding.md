TODO: add title & stuff

It'd be kind of cool to work through & then write about the following:
- conceptualize each path to traverse as a lazy path
- the next step of the path is determined by the path finding algorithm
- multiple next step possibilities are `mapcat`ed, so you evolve
	1 (start node path)
	2 ( [start node -> adjacent_a -> lazy path], [start node -> adjacent_b -> lazy path])
 	3 ...
- If this is then a stream of paths, each one lazy, we can apply transducers to do things that would otherwise be difficult
  such as stop searching arbitrarily, ignore paths with arbitrary criteria (that transfer BTC -> BTC -> BTC for example)

Alternatively, it could make sense to expand on the lazy graph function.
So, the graph function would generate adjacents given path & node.
I.e. ({node}, node) => {node} instead of node => {node}.
Then the predicate logic would apply to the first argument of the adjacent generator. 
