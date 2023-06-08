<script lang="ts">
    import { onMount } from "svelte";
    import Matter, { Body, Constraint, Common, World } from "matter-js";
    import { Timestamp, collection, doc, getFirestore, onSnapshot } from "firebase/firestore";
    import { initializeApp } from "firebase/app";

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

    const firebaseConfig = {
        apiKey: "AIzaSyC556xEWQ6bn9dXxnND7zn22K_YMjqGG5k",
        authDomain: "ai-profielwerkstuk.firebaseapp.com",
        projectId: "ai-profielwerkstuk",
        storageBucket: "ai-profielwerkstuk.appspot.com",
        messagingSenderId: "458512798569",
        appId: "1:458512798569:web:edd96f8243b2c18a54e3f8"
    };

    const app = initializeApp (firebaseConfig);
    const firestore = getFirestore(app);

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

    const genomes: {genome: Genome, uploadDate: Timestamp}[] = []

    onMount(async () => {
        onSnapshot(collection(firestore, "genomes"), (doc) => {
            doc.docChanges().forEach(change => {
                if (change.type === "added") {
                    genomes.push(change.doc.data().genome);
                } else if (change.type === "modified") {
                    genomes[genomes.findIndex(v => v.CarInstance.id === change.doc.id)] = change.doc.data() as Genome;
                }
            });
        });

        const res = await fetch("/genomes/bram.json");
        let GENOME = (await res.json()) as Genome;

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

        let allBodies: (Body | Constraint)[] = [];
        function renderFunction() {
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

            allBodies.push(...Object.values(nodes));
            allBodies.push(...lines);

            Composite.add(engine.world, allBodies);

            Render.run(render);

            const runner = Runner.create();
            Runner.run(runner, engine);

            function applyRandomForce(bodies: Body[]) {
                const forceMagnitude = Common.random(0.03, 0.08);
                const forceAngle = Common.random(0, 2 * Math.PI);

                const forceX = Math.cos(forceAngle) * forceMagnitude;
                const forceY = Math.sin(forceAngle) * forceMagnitude;

                const body = bodies[Math.floor(Math.random() * bodies.length)];
                Body.applyForce(body, body.position, { x: forceX, y: forceY });

                const interval = Common.random(100, 500);
                setTimeout(() => applyRandomForce(bodies), interval);
            }
            applyRandomForce([...Object.values(nodes)]);
        }

        renderFunction();

        addEventListener("keypress", (e) => {
            if (e.key === "l") {
                const input = JSON.parse(prompt("Paste the brain:")!);
                GENOME = input as unknown as Genome;

                World.clear(engine.world, false);
                Engine.clear(engine);

                allBodies = [];

                renderFunction();
            }

            if (e.key === "f") {
                allBodies.forEach((v) => {
                    // @ts-ignore This is the way to do it
                    if (v.type === "body") v.isStatic = !v.isStatic;
                });
            }
        });
    });
</script>

<style>
    :global(*) {
        padding: 0px;
        margin: 0px;
        overflow: hidden;
    }
</style>
