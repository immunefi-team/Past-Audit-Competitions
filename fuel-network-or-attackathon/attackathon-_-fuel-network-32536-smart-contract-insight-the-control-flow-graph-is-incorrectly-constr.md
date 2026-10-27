# Attackathon \_ Fuel Network 32536 - \[Smart Contract - Insight] The control flow graph is incorrectly

Submitted on Tue Jun 25 2024 18:34:32 GMT-0400 (Atlantic Standard Time) by @Schnilch for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32536

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Compiler fails to deliver promised returns

## Description

## Brief/Intro

When the control flow graph is constructed for return path analysis, the mistake is made that only the "first level" of the code is examined. This means that, for example, if an if statement occurs in the code, the if statement is included in the graph, but the code within the if statement is not further analyzed. As a result, the return path analysis that uses this graph cannot function correctly since it is possible that there is a return in the if statement. However, this currently has no real impact as everything is caught by the type check.

## Vulnerability Details

After the code has been type-checked, some further analyses are performed, including the return path analysis which checks whether all possible paths in the code provide a return value (See 1. reference). For this, a control flow graph is built (See 2. reference). In doing so, the function connect\_node is used, which incorrectly handles expressions (See 3. reference). connect\_node handles all expressions except for return expressions as follows:

```rust
        ty::TyAstNodeContent::Expression(ty::TyExpression { .. }) => {
            let entry = graph.add_node(ControlFlowGraphNode::from_node(node));
            // insert organizational dominator node
            // connected to all current leaves
            for leaf in leaves {
                graph.add_edge(*leaf, entry, "".into());
            }
            Ok(NodeConnection::NextStep(vec![entry]))
        }
```

In this process, we see that regardless of the type of expression (except for return expressions), it is simply added to the graph and connected to the leaves. However, there are expressions such as if statements that contain code. This code, as seen in the above code snippet, is not added to the graph. The issue with this approach is that the code may contain return statements that are not accounted for. Something similar can happen when the last expression in a function, which is supposed to return something, also contains code:

```rust
        ty::TyAstNodeContent::Expression(ty::TyExpression {
            expression: ty::TyExpressionVariant::Return(..),
            ..
        })
        | ty::TyAstNodeContent::Expression(ty::TyExpression {
            expression: ty::TyExpressionVariant::ImplicitReturn(..),
            ..
        }) => {
            let this_index = graph.add_node(ControlFlowGraphNode::from_node(node));
            for leaf_ix in leaves {
                graph.add_edge(*leaf_ix, this_index, "".into());
            }
            Ok(NodeConnection::Return(this_index))
        }
```

In this case, the expression could again be, for example, an if statement that contains code. Here again, this code would not be added to the graph. Only the if statement itself would be added to the graph as a return expression, without checking whether something is actually returned within the if statement.

## Impact Details

Although due to this bug the return path analysis did not work correctly, currently it has no real consequences because if a return value were missing, it would be detected by the type check. If no value is returned from a path in the function that should return something, it is simply recognized as returning a value with the wrong type. However, this bug could have consequences if, for example, the compiler is further developed and the type check no longer catches this error, relying instead on the return path analysis. Additionally, at the moment, the return path analysis does not return errors as it should.

## References

1. <https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/lib.rs#L1055-L1059>
2. <https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/lib.rs#L1069>
3. <https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/control\\_flow\\_analysis/analyze\\_return\\_paths.rs#L140-L189>

## Proof of concept

## Proof of Concept

To show that the control flow graph is incorrectly constructed, it is best to visualize it, as this allows you to see most clearly that a part of the code is missing in the graph. To do this, the graph must first be output to a file after it has been visualized. For this, the following line must be inserted into sway-core/src/lib.rs:

```diff
1069:     let graph = ControlFlowGraph::construct_return_path_graph(engines, &module.all_nodes);
+ 1070:     graph.clone().unwrap().visualize(engines, Option::Some("return_graph".to_owned()), None);
1071:     match graph {
```

Now the following POC can be inserted into the file sway-core/src/lib.rs and started with this command: `cargo test -p sway-core poc -- --nocapture`

```rust
#[test]
fn poc() {
    //Setup
    let handler = Handler::default();
    let engines = Engines::default();

    let programs: Option<(parsed::ParseProgram, LexedProgram)> = match parse( //The code is parsed so that it can be used in the next step for the type check.
        //That is the code that gets compiled
        r#"
        script;

        fn main() -> u64 {
            if false { //This logic is there to show later that it does not appear in the control flow graph
                let mut i = 0;
                while __lt(i, 10) {
                    i = __add(i, 1);
                }
            } else {
                return 10;
            }
        }
        "#.into(),
        &handler,
        &engines,
        None
    ) {
        Err(err) => {
            println!("error: {:?}", err);
            None
        },
        Ok((lexed, program)) => {
            Some((program, lexed))
        }
    };

    let (mut parse_program, _) = programs.unwrap();

    let ty_program = parsed_to_ast( //In this function, the type check is now performed, and the return path analysis as well
        &handler, 
        &engines, 
        &mut parse_program, 
        namespace::Root { module: namespace::Module::default() },
        None,
        "test-project", 
        None
    );

    let fail = |handler: Handler| {
        let (errors, warnings) = handler.consume();
        println!("errors: {:#?}", errors);
    };

    /*
    Here, the errors from the handler are read and printed. In this case, you will see some errors indicating that the wrong type is returned, 
    but no error from the return path analysis that there is a missing return value.
     */
    if handler.has_errors()  {
        return fail(handler);
    }
}
```

Now you should see the errors that were caught during the type check but no errors from the return path analysis in the terminal. Additionally, there should now be a file sway-core/return\_graph where the graph is located. This graph can now be copied and visualized, for example, on this website: <https://dreampuf.github.io/GraphvizOnline/> The code for the graph should look like this:

```
digraph {
    0 [  label = "function declaration (main)" ]
    1 [  label = "\"main\" fn exit" ]
    2 [  label = "implicit return if exp (()) (())" ]
    0 -> 2 [ label = ""]
    2 -> 1 [ label = "return"]
}
```

Based on the graph, it is clear that while the if statement appears as an implicit return, the code within the if statement that would actually perform the return is not included. Since the return path analysis iterates over this graph later, it will not check whether the if statement actually returns anything.
