<script>
	import { onMount } from "svelte";
	import { makeDB } from "./duck.js";
	import initSqlJs from "sql.js";
	import model1 from "./cube.obj?raw";
	import model2 from "./suzanne.obj?raw";
	// or if you are in a browser:
	// const initSqlJs = window.initSqlJs;

	let clockwise = $state(null);
	let vertices = $state([]);
	let edges = $state([]);
	let faces = $state([]);
	let camera = $state({});
	let transform = $state([]);
	let viewport = $state({ width: 0, height: 0 });
	let statements = {};

	const schemaCreation = [
		`CREATE TABLE camera (id integer primary key AUTOINCREMENT, tx real NOT NULL, ty real NOT NULL, tz real NOT NULL, near real NOT NULL, far real NOT NULL, fov real NOT NULL, aspect real NOT NULL) STRICT`,
		`CREATE TABLE viewport (id integer primary key AUTOINCREMENT, width REAL NOT NULL, height REAL NOT NULL, camera_id integer NOT NULL REFERENCES camera(id) ON DELETE CASCADE) STRICT`,
		`CREATE TABLE model (id integer primary key AUTOINCREMENT ) STRICT`,
		`CREATE TABLE model_transform (id integer primary key AUTOINCREMENT, model_id integer NOT NULL UNIQUE REFERENCES model(id) ON DELETE CASCADE, tx real NOT NULL DEFAULT 0.0, ty real NOT NULL DEFAULT 0.0, tz real NOT NULL DEFAULT 0.0, rx real NOT NULL DEFAULT 0.0, ry real NOT NULL DEFAULT 0.0, rz real NOT NULL DEFAULT 0.0, sx real NOT NULL DEFAULT 1.0, sy real NOT NULL DEFAULT 1.0, sz real NOT NULL DEFAULT 1.0) STRICT`,
		`CREATE TABLE vertex (id integer primary key AUTOINCREMENT, model_id integer NOT NULL REFERENCES model(id) ON DELETE CASCADE, x real NOT NULL, y real NOT NULL, z real NOT NULL) STRICT`,
		`CREATE TABLE edge (id integer primary key AUTOINCREMENT, model_id integer NOT NULL REFERENCES model(id) ON DELETE CASCADE, start_vertex_id integer NOT NULL REFERENCES vertex(id) ON DELETE CASCADE, end_vertex_id integer NOT NULL REFERENCES vertex(id) ON DELETE CASCADE) STRICT`,
		`CREATE TABLE face (id integer primary key AUTOINCREMENT, model_id integer NOT NULL REFERENCES model(id) ON DELETE CASCADE) STRICT`,
		`CREATE TABLE face_vertex (id integer primary key AUTOINCREMENT, model_id integer NOT NULL REFERENCES model(id) ON DELETE CASCADE, face_id integer NOT NULL REFERENCES face(id) ON DELETE CASCADE, vertex_id integer NOT NULL REFERENCES vertex(id) ON DELETE CASCADE, sort integer) STRICT`,
		`CREATE VIEW model_vertex AS SELECT v.id AS vertex_id, m.id AS model_id, ((cos(COALESCE(mt.ry,0))*cos(COALESCE(mt.rz,0))*(v.x*COALESCE(mt.sx, 1)) + (-cos(COALESCE(mt.ry,0))*sin(COALESCE(mt.rz,0)))*(v.y*COALESCE(mt.sy, 1)) + sin(COALESCE(mt.ry,0))*(v.z*COALESCE(mt.sz, 1)))-COALESCE(mt.tx, 0)) AS x, ((((cos(COALESCE(mt.rx,0))*sin(COALESCE(mt.rz,0)) + sin(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*cos(COALESCE(mt.rz,0)))*(v.x*COALESCE(mt.sx, 1)) + (cos(COALESCE(mt.rx,0))*cos(COALESCE(mt.rz,0)) - sin(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*sin(COALESCE(mt.rz,0)))*(v.y*COALESCE(mt.sy, 1)) + (-sin(COALESCE(mt.rx,0))*cos(COALESCE(mt.ry,0)))*(v.z*COALESCE(mt.sz, 1))))-COALESCE(mt.ty, 0)) AS y, (((sin(COALESCE(mt.rx,0))*sin(COALESCE(mt.rz,0)) - cos(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*cos(COALESCE(mt.rz,0)))*(v.x*COALESCE(mt.sx, 1)) + (sin(COALESCE(mt.rx,0))*cos(COALESCE(mt.rz,0)) + cos(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*sin(COALESCE(mt.rz,0)))*(v.y*COALESCE(mt.sy, 1)) + (cos(COALESCE(mt.rx,0))*cos(COALESCE(mt.ry,0)))*(v.z*COALESCE(mt.sz, 1)))-COALESCE(mt.tz, 0)) AS z FROM vertex v INNER JOIN model m ON m.id = v.model_id INNER JOIN model_transform mt ON mt.model_id = m.id`,
		`CREATE VIEW camera_vertex AS SELECT v.vertex_id AS vertex_id, c.id AS camera_id, (v.x-c.tx) AS x, (v.y-c.ty) AS y, (v.z-c.tz) AS z FROM model_vertex v, camera c`,
		`CREATE VIEW perspective_vertex AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/tan(c.fov/2)*c.aspect) AS x, (v.y/tan(c.fov/2)) AS y, (-v.z*((c.far+c.near)/(c.far-c.near))-((2*c.far*c.near)/(c.far-c.near))) AS z, -v.z as w FROM camera_vertex v INNER JOIN camera c ON c.id = v.camera_id`,
		`CREATE VIEW ndc_vertex AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/v.w) AS x, (v.y/v.w) AS y, (v.z/v.w) AS z, v.w AS w FROM perspective_vertex v`,

		`CREATE VIEW ndc_edge AS SELECT edge.id AS id, a.camera_id AS camera_id, (a.x/a.w) AS x1, (a.y/a.w) AS y1, (a.z/a.w) AS z1, (a.w/a.w) AS w1, (b.x/b.w) AS x2, (b.y/b.w) AS y2, (b.z/b.w) AS z2, b.w AS w2 FROM edge INNER JOIN perspective_vertex a ON a.vertex_id = edge.start_vertex_id INNER JOIN perspective_vertex b ON b.vertex_id = edge.end_vertex_id WHERE a.camera_id = b.camera_id`,
		//`CREATE VIEW ndc_face AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/v.w) AS x, (v.y/v.w) AS y, (v.z/v.w) AS z, v.w AS w FROM perspective_vertex v`,

		`CREATE VIEW projected_vertex AS SELECT vp.id AS viewport_id, v.vertex_id AS vertex_id, v.camera_id AS camera_id, vp.width * (0.5 + 0.5*v.x) AS x, vp.height * (0.5 + 0.5*v.y) AS y, v.z AS z, v.w AS w FROM ndc_vertex v INNER JOIN viewport vp ON vp.camera_id = v.camera_id WHERE abs(v.x) < 1 AND  abs(v.y) < 1  AND abs(v.z) < 1`,
		`CREATE VIEW projected_edge AS SELECT vp.id AS viewport_id, e.id AS edge_id, e.camera_id AS camera_id, vp.width * (0.5 + 0.5*e.x1) AS x1, vp.height * (0.5 + 0.5*e.y1) AS y1, e.z1 AS z1, e.w1 AS w1, vp.width * (0.5 + 0.5*e.x2) AS x2, vp.height * (0.5 + 0.5*e.y2) AS y2, e.z2 AS z2, e.w2 AS w2 FROM ndc_edge e INNER JOIN viewport vp ON vp.camera_id = e.camera_id WHERE abs(e.x1) < 1 AND  abs(e.y1) < 1  AND abs(e.z1) < 1 AND  abs(e.x2) < 1 AND  abs(e.y2) < 1  AND abs(e.z2) < 1`,

		`CREATE VIEW face_vertex_neighbor AS SELECT face_vertex.*,
			         LEAD(face_vertex.vertex_id) OVER w AS next_vertex_id,
			         LAG(face_vertex.vertex_id) OVER w AS prev_vertex_id
			  FROM face_vertex
			  WINDOW w AS (PARTITION BY face_vertex.face_id ORDER BY sort)`,
		`CREATE VIEW face_vertex_neighbor_wrapped_helper AS 
			 SELECT fv.*,
					   fvn.next_vertex_id AS next_vertex_id,
					   fvn.prev_vertex_id AS prev_vertex_id,
			           FIRST_VALUE(fvn.vertex_id) OVER (PARTITION BY fvn.face_id ORDER BY fvn.sort) AS first_id,
			           FIRST_VALUE(fvn.vertex_id) OVER (PARTITION BY fvn.face_id ORDER BY -fvn.sort) AS last_id
			    FROM face_vertex_neighbor fvn 
			    INNER JOIN face_vertex fv ON fv.id = fvn.id`,
		`CREATE VIEW face_vertex_neighbor_wrapped AS 
			SELECT
					 helper.face_id AS face_id,
					 helper.vertex_id AS vertex_id, 
			         COALESCE(next_vertex_id, first_id) AS id_next,
			         COALESCE(prev_vertex_id, last_id) AS id_prev
			  FROM face_vertex_neighbor_wrapped_helper helper`,
		`CREATE VIEW face_projected_signed_area AS SELECT 
			fvw.face_id AS face_id,
			 SUM((v1.x * v2.y) - (v2.x * v1.y)) AS area_twice
			FROM face_vertex_neighbor_wrapped fvw 
			INNER JOIN projected_vertex v1 ON v1.vertex_id = fvw.vertex_id
			INNER JOIN projected_vertex v2 ON v2.vertex_id = fvw.id_next
			GROUP BY fvw.face_id`,
		`CREATE VIEW projected_face AS SELECT fv.model_id as model_id, v.viewport_id AS viewport_id, IF(MAX(face_area.area_twice) > 0, 'blue', 'red') AS clockcolor, IF(MAX(face_area.area_twice) > 0, 0.5, 1) AS opacity, MAX(face_area.area_twice) > 0 AS clockwise, face.id AS face_id, GROUP_CONCAT(CONCAT(v.x, ', ', v.y), ' ') AS points FROM face INNER JOIN face_projected_signed_area face_area ON face_area.face_id = face.id INNER JOIN face_vertex fv ON fv.face_id = face.id INNER JOIN projected_vertex v ON v.vertex_id = fv.vertex_id GROUP BY fv.face_id, fv.model_id, face.id, v.viewport_id ORDER BY MAX(face_area.area_twice) ASC, MIN(v.z) DESC`,
		`INSERT INTO camera(id,tx,ty,tz, near, far, fov, aspect) VALUES(1, 0, 0, 50, 0.1, 100, 1, 1)`,
		`INSERT INTO viewport(id, camera_id, width,height) VALUES(1, 1, 1, 1)`,

		"PRAGMA foreign_keys = ON",
	];

	const statementTemplates = (statements = {
		cameraSelect: "SELECT * FROM camera",
		transformSelect: "SELECT * FROM model_transform ORDER BY id DESC",
		viewportSelect: "SELECT * FROM viewport",
		vertexSelect:
			"SELECT vertex_id AS id, x, y, z, w FROM projected_vertex WHERE viewport_id = $viewport",
		edgeSelect:
			"SELECT * FROM projected_edge WHERE viewport_id = $viewport",
		faceSelect:
			"SELECT * FROM projected_face WHERE viewport_id = $viewport AND ($clockwise = clockwise) IS DISTINCT FROM FALSE ORDER BY ($clockwise <> clockwise) ASC",
	});

	let db;

	function importModel(
		db,
		model,
		sx = 1,
		sy = 1,
		sz = 1,
		{ tx = 0, ty = 0, tz = 0 } = {},
	) {
		const lines = model.split("\n");

		const insertModel = db.prepare(
			"INSERT INTO model DEFAULT VALUES  RETURNING id",
		);
		const insertModelTransform = db.prepare(
			"INSERT INTO model_transform(model_id,tx,ty,tz) VALUES(?,?,?,?) RETURNING  id",
		);
		const insertVertex = db.prepare(
			"INSERT INTO vertex (model_id, x,y,z) VALUES(?,?,?,?) RETURNING id",
		);
		const insertFace = db.prepare(
			"INSERT INTO face (model_id) VALUES(?) RETURNING id",
		);
		const insertFaceVertex = db.prepare(
			"INSERT INTO face_vertex (model_id, face_id, vertex_id, sort) VALUES(?,?,?, ?) RETURNING id",
		);
		let modelId = null;
		const vertexIds = [];

		for (const line of lines) {
			if (line.startsWith("o ")) {
				insertModel.step();
				[modelId] = insertModel.get();
				insertModelTransform.bind([modelId, tx, ty, tz]);
				insertModelTransform.step();
				const [modelTransformId] = insertModelTransform.get();
			} else if (line.startsWith("v ")) {
				const [x, y, z] = line
					.slice("v ".length)
					.split(/\s+/, 3)
					.map(parseFloat);
				insertVertex.bind([modelId, sx * x, sy * y, sz * z]);
				insertVertex.step();
				const [vertexId] = insertVertex.get();
				vertexIds.push(vertexId);
			} else if (line.startsWith("f ")) {
				const vertices = line
					.slice("v ".length)
					.split(/\s+/)
					.filter((x) => x)
					.map((v) =>
						v
							.split("/")
							.filter((x) => x)
							.map((x) => parseInt(x, 10)),
					);

				if (vertices.length === 3) {
					insertFace.bind([modelId]);
					insertFace.step();
					const [faceId] = insertFace.get();
					let i = 0;
					for (const [v] of vertices) {
						insertFaceVertex.bind([
							modelId,
							faceId,
							vertexIds[v - 1],
							i++,
						]);
						insertFaceVertex.step();
						const [faceVertexId] = insertFaceVertex.get();
					}
				} else if (vertices.length === 4) {
					for (let verts of [
						[vertices[0], vertices[1], vertices[2]],
						[vertices[0], vertices[2], vertices[3]],
					]) {
						insertFace.bind([modelId]);
						insertFace.step();
						const [faceId] = insertFace.get();
						let i = 0;
						for (const [v] of verts) {
							insertFaceVertex.bind([
								modelId,
								faceId,
								vertexIds[v - 1],
								i++,
							]);
							insertFaceVertex.step();
							const [faceVertexId] = insertFaceVertex.get();
						}
					}
				}
			}
		}

		insertModel.free();
		insertModelTransform.free();
		insertVertex.free();
		insertFace.free();
		insertFaceVertex.free();
	}

	onMount(async () => {
		const SQL = await initSqlJs({
			// Required to load the wasm binary asynchronously. Of course, you can host it wherever you want
			// You can omit locateFile completely when running in node
			locateFile: (file) => `https://sql.js.org/dist/${file}`,
		});
		db = new SQL.Database();

		const duck = await makeDB();

		for (const cmd of schemaCreation) {
			if (cmd.startsWith("PRAGMA")) {
				continue;
			}
			await duck.query(
				cmd
					.replace(
						/CREATE TABLE (\S+) \(id integer primary key AUTOINCREMENT/,
						"CREATE SEQUENCE seq_$1_id START 1; CREATE TABLE $1 (id integer primary key DEFAULT(nextval('seq_$1_id'))",
					)
					.replace(/\) STRICT$/, ")")
					.replace(/ ON DELETE (CASCADE|SET NULL|SET DEFAULT)/g, "")
					.replace(/ ON UPDATE (CASCADE|SET NULL|SET DEFAULT)/g, ""),
			);
		}

		db.run(schemaCreation.join(";\n"));

		statements = Object.fromEntries(
			Object.entries(statementTemplates).map(([k, v]) => [
				k,
				db.prepare(v),
			]),
		);
		setViewport(db, window.innerWidth, window.innerHeight);

		importModel(db, model1, 4, 4, 4, { tx: 10 });

		update(db);

		return () => {
			vertices = undefined;
			edges = undefined;
			faces = undefined;

			Object.values(statements).forEach((s) => s.free());
		};
	});

	function setCamera(db, { x, y, z, near, far, fov }) {
		db.run(
			"UPDATE camera SET tx=?,ty=?,tz=?,near=?,far=?, fov=? WHERE id = 1",
			[x, y, z, near, far, fov],
		);
		update(db);
	}

	function setTransform(db, model, { tx, ty, tz, rx, ry, rz, sx, sy, sz }) {
		db.run(
			"UPDATE model_transform SET tx=?, ty=?, tz=?, rx=?, ry=?, rz=?, sx=?, sy=?, sz=? WHERE model_id = ?",
			[tx, ty, tz, rx, ry, rz, sx, sy, sz, model],
		);
		update(db);
	}

	function deleteModel(db, model_id) {
		db.run("DELETE FROM model WHERE id = ?", [model_id]);
		update(db);
	}

	function setViewport(db, w, h) {
		db.run("UPDATE viewport SET width=?,height=? WHERE id = 1", [w, h]);
		db.run("UPDATE camera SET aspect=? WHERE id = 1", [h / w]);
		update(db);
	}

	function update(db) {
		if (db) {
			// const test = db.prepare("SELECT * FROM face_vertex");
			// while (test.step()) {
			// 	console.log(test.getAsObject());
			// }
			{
				if (statements.cameraSelect.step()) {
					camera = statements.cameraSelect.getAsObject();
				}
			}
			{
				const t = [];
				while (statements.transformSelect.step()) {
					t.push(statements.transformSelect.getAsObject());
				}
				transform = t;
			}

			{
				if (statements.viewportSelect.step()) {
					viewport = statements.viewportSelect.getAsObject();
				}
			}

			// Prepare an sql statement
			{
				const rows = [];
				statements.vertexSelect.bind({ $viewport: 1 });
				while (statements.vertexSelect.step()) {
					//
					rows.push(statements.vertexSelect.getAsObject());
				}

				vertices = rows;
			}

			{
				const rows = [];
				statements.edgeSelect.bind({ $viewport: 1 });
				while (statements.edgeSelect.step()) {
					//
					rows.push(statements.edgeSelect.getAsObject());
				}
				edges = rows;
			}

			{
				const rows = [];
				statements.faceSelect.bind({
					$viewport: 1,
					$clockwise: clockwise,
				});
				while (statements.faceSelect.step()) {
					const face = statements.faceSelect.getAsObject();

					rows.push(face);
				}
				faces = Object.values(rows);
			}
		}
	}
</script>

<div class="app">
	<div>
		<h1>SQLite 3D Renderer</h1>
		<p>This is a demo of projecting 3d triangles via SQL queries.</p>
		<p>
			Think <code
				>SELECT project(...) as x, project(...) as y FROM model, vertex,
				camera, transform WHERE clockwise AND clipped IN BETWEEN -1 AND
				1</code
			>
		</p>
	</div>
	<div class="overlay">
		<h2>Backface Culling</h2>
		<form
			onchange={(evt) => {
				update(db);
			}}
		>
			<label
				><input type="radio" bind:group={clockwise} value={null} /> Both</label
			>
			<label
				><input type="radio" bind:group={clockwise} value={1} /> Clockwise</label
			>
			<label
				><input type="radio" bind:group={clockwise} value={0} />
				CounterClockwise</label
			>
		</form>
		<h2>Camera</h2>
		<form
			oninput={(evt) => {
				setCamera(
					db,
					Object.fromEntries(new FormData(evt.currentTarget)),
				);
			}}
		>
			<div class="slider-group">
				<label
					><span>x:</span>
					<input
						name="x"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={camera.tx}
					/></label
				>
				<label
					><span>y:</span>
					<input
						name="y"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={camera.ty}
					/></label
				>
				<label
					><span>z:</span>
					<input
						name="z"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={camera.tz}
					/></label
				>
				<label
					><span>near:</span>
					<input
						name="near"
						step="0.0001"
						type="range"
						min="0.1"
						max="100"
						value={camera.near}
					/></label
				>
				<label
					><span>far:</span>
					<input
						name="far"
						step="0.0001"
						type="range"
						min="0.1"
						max="100"
						value={camera.far}
					/></label
				>
				<label
					><span>fov:</span>
					<input
						name="fov"
						step="0.0001"
						type="range"
						min="0.1"
						max="3"
						value={camera.fov}
					/></label
				>
			</div>
		</form>

		<div>
			<button
				type="button"
				onclick={(e) => {
					importModel(db, model1, 4, 4, 4, {
						tx: 10 - 10 * transform.length,
					});
					update(db);
				}}>Add Cube</button
			>

			<button
				type="button"
				onclick={(e) => {
					importModel(db, model2, -8, -8, 8, {
						tx: 10 - 10 * transform.length,
					});
					update(db);
				}}>Add Suzanne</button
			>
		</div>

		{#each transform as t}
			<h2>Model #{t.model_id}</h2>
			<button type="button" onclick={(e) => deleteModel(db, t.model_id)}
				>Delete</button
			>
			<form
				oninput={(evt) => {
					setTransform(
						db,
						t.model_id,
						Object.fromEntries(
							[...new FormData(evt.currentTarget)].map(
								([k, v]) => [k, parseFloat(v)],
							),
						),
					);
				}}
			>
				<div class="slider-grid">
					<label
						><span>tx:</span>
						<input
							name="tx"
							step="0.0001"
							type="range"
							min="-60"
							max="60"
							value={t.tx}
						/></label
					>
					<label
						><span>ty:</span>
						<input
							name="ty"
							step="0.0001"
							type="range"
							min="-60"
							max="60"
							value={t.ty}
						/></label
					><label
						><span>tz:</span>
						<input
							name="tz"
							step="0.0001"
							type="range"
							min="-60"
							max="60"
							value={t.tz}
						/></label
					>
					<label
						><span>rx:</span>
						<input
							name="rx"
							step="0.0001"
							type="range"
							min="-3"
							max="3"
							value={t.rx}
						/></label
					>
					<label
						><span>ry:</span>
						<input
							name="ry"
							step="0.0001"
							type="range"
							min="-3"
							max="3"
							value={t.ry}
						/></label
					><label
						><span>rz:</span>
						<input
							name="rz"
							step="0.0001"
							type="range"
							min="-3"
							max="3"
							value={t.rz}
						/></label
					>

					<label
						><span>sx:</span>
						<input
							name="sx"
							step="0.0001"
							type="range"
							min="-4"
							max="4"
							value={t.sx}
						/></label
					>
					<label
						><span>sy:</span>
						<input
							name="sy"
							step="0.0001"
							type="range"
							min="-4"
							max="4"
							value={t.sy}
						/></label
					><label
						><span>sz:</span>
						<input
							name="sz"
							step="0.0001"
							type="range"
							min="-4"
							max="4"
							value={t.sz}
						/></label
					>
				</div>
			</form>
		{/each}

		<h2>DB Schema</h2>
		<div style="width: 100%;  overflow: auto;">
			{#each schemaCreation as table}
				<pre>{table}</pre>
			{/each}

			{#each statementTemplates as _, select}
				<pre>{select}</pre>
			{/each}
		</div>
	</div>

	<svg class="viewport" viewBox="0 0 {viewport.width} {viewport.height}">
		{#each faces as f}
			<polygon
				data-model={f.model_id}
				stroke-linejoin="round"
				stroke-linecap="round"
				fill={f.clockcolor}
				opacity={f.opacity}
				stroke={"black"}
				stroke-width="1"
				points={f.points}
				stroke-dasharray={f.clockwise ? "none" : "10 10"}
			></polygon>
		{/each}

		<!-- {#each vertices as v}
			<circle r="3" cx={v.x} cy={v.y}></circle>
		{/each} -->
	</svg>
</div>

<style>
	h2 {
		margin: 0;
		font-size: 1em;
	}
	:global(body) {
		margin: 0;
	}
	svg {
		position: fixed;
		left: 0;
		right: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}
	.app {
		display: grid;
		padding: 1em;
		grid-template-columns: repeat(auto-fill, minmax(32em, 1fr));
		grid-template-rows: 1fr 1fr 1fr;
		position: absolute;
		gap: 1em;
		width: 100%;
		height: 100%;
		box-sizing: border-box;
	}

	.overlay {
		grid-area: 1/1 / 4 /1;
		background: #aaa3;
		padding: 1em;
		gap: 1em;
		display: grid;
		align-content: start;
	}

	.viewport {
		grid-area: 1 / 1 / -1 / -1;
	}

	.slider-group {
		display: grid;
		grid-template-columns: auto 1fr;
		gap: 1ex;
		align-items: baseline;
	}

	.slider-group > label {
		display: contents;
	}

	.slider-grid {
		display: grid;
		grid-template-columns: auto 1fr auto 1fr auto 1fr;
		gap: 1ex;
		align-items: baseline;
	}

	.slider-grid > label {
		display: contents;
	}
</style>
