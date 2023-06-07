<script lang="ts">
    import { onMount } from "svelte";
    import Matter, { Body, Constraint, Common } from "matter-js";

    const SETTINGS = {
        neuronSize: 20,
        colourKeys: {
            INPUT: {
                fillStyle: "#0000FF",
                strokeStyle: "#FFFFFF00",
            },
            OUTPUT: {
                fillStyle: "#FF0000",
                strokeStyle: "#FFFFFF00",
            },
            HIDDEN: {
                fillStyle: "#FFFFFF00",
                strokeStyle: "#FFFFFF",
            },
        },
    };

    type Node = { type: "INPUT" | "OUTPUT" | "HIDDEN"; id: string; active: boolean };
    type Connection = {
        inputNode: Node;
        outputNode: Node;
        weight: number;
        active: boolean;
    };
    type Genome = {
        nodes: Node[];
        connections: Connection[];
        fitness: number;
    };

    onMount(async () => {
        const res = await fetch("/genomes/bram.json");
        const GENOME = (await res.json()) as Genome;

        const Engine = Matter.Engine,
            Render = Matter.Render,
            Runner = Matter.Runner,
            Bodies = Matter.Bodies,
            Composite = Matter.Composite;

        const engine = Engine.create();
        engine.gravity.y = 0;

        const render = Render.create({
            element: document.body,
            engine: engine,
            options: {
                width: window.innerWidth,
                height: window.innerHeight,
                wireframes: false,
            },
        });

        const ground = Bodies.rectangle(window.innerWidth / 2, window.innerHeight, window.innerWidth, 50, { isStatic: true });
        const ceiling = Bodies.rectangle(window.innerWidth / 2, 0, window.innerWidth, 50, { isStatic: true });
        const leftWall = Bodies.rectangle(0, window.innerHeight / 2, 50, window.innerHeight, { isStatic: true });
        const rightWall = Bodies.rectangle(window.innerWidth, window.innerHeight / 2, 50, window.innerHeight, { isStatic: true });
        Composite.add(engine.world, [ground, ceiling, leftWall, rightWall]);

        const nodes: { [key: string]: Body } = {};
        const lines: Constraint[] = [];

        for (const node of GENOME.nodes) {
            const Y = Math.floor(0.5 * window.innerHeight);
            const X = Math.floor(0.5 * window.innerWidth);

            nodes[node.id] = Bodies.circle(X, Y, SETTINGS.neuronSize, {
                frictionAir: 1,
                render: {
                    fillStyle: node.active ? SETTINGS.colourKeys[node.type].fillStyle : SETTINGS.colourKeys[node.type].fillStyle,
                    strokeStyle: node.active ? SETTINGS.colourKeys[node.type].strokeStyle : "red",
                    lineWidth: 2,
                },
            });
        }

        for (const connection of GENOME.connections) {
            const line = Constraint.create({
                bodyA: nodes[connection.inputNode.id],
                bodyB: nodes[connection.outputNode.id],
                length: Math.abs(connection.weight) * 650,
                stiffness: 0.8,
                render: {
                    strokeStyle: `#FFFFFF`,
                    lineWidth: Math.abs(connection.weight) * 1,
                },
            });

            lines.push(line);
        }

        const allBodies = [];
        allBodies.push(...Object.values(nodes));
        allBodies.push(...lines);

        Composite.add(engine.world, allBodies);

        Render.run(render);

        const runner = Runner.create();
        Runner.run(runner, engine);

        function applyRandomForce(bodies: Body[]) {
            // Generate random force values
            const forceMagnitude = Common.random(0.03, 0.08); // Random magnitude between 0.005 and 0.02
            const forceAngle = Common.random(0, 2 * Math.PI); // Random angle between 0 and 2 * PI

            // Calculate force components
            const forceX = Math.cos(forceAngle) * forceMagnitude;
            const forceY = Math.sin(forceAngle) * forceMagnitude;

            const body = bodies[Math.floor(Math.random() * bodies.length)];
            // Apply the force to the body
            Body.applyForce(body, body.position, { x: forceX, y: forceY });

            // Schedule the next random force
            const interval = Common.random(100, 500); // Random interval between 500 and 2000 milliseconds
            setTimeout(() => applyRandomForce(bodies), interval);
        }
        applyRandomForce([...Object.values(nodes)]);
    });
</script>

<style>
    :global(*) {
        padding: 0px;
        margin: 0px;
        overflow: hidden;
    }
</style>
