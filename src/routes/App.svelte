<script>
	import { onMount } from "svelte";
	import initSqlJs from "sql.js";
	// or if you are in a browser:
	// const initSqlJs = window.initSqlJs;

	let clockwise = $state(null);
	let vertices = $state(undefined);
	let edges = $state(undefined);
	let faces = $state(undefined);
	let camera = $state({});
	let transform = $state({});
	let viewport = $state({ width: 0, height: 0 });
	let statements = {};

	let db;

	onMount(async () => {
		const SQL = await initSqlJs({
			// Required to load the wasm binary asynchronously. Of course, you can host it wherever you want
			// You can omit locateFile completely when running in node
			locateFile: (file) => `https://sql.js.org/dist/${file}`,
		});
		db = new SQL.Database();
		let sqlstr = [
			`CREATE TABLE camera (id integer primary key, tx real NOT NULL, ty real NOT NULL, tz real NOT NULL, near real NOT NULL, far real NOT NULL, fov real NOT NULL, aspect real NOT NULL) STRICT`,
			`CREATE TABLE viewport (id integer primary key, width REAL NOT NULL, height REAL NOT NULL, camera_id integer NOT NULL REFERENCES camera(id)) STRICT`,
			`CREATE TABLE model (id integer primary key) STRICT`,
			`CREATE TABLE model_transform (id integer primary key, model_id integer NOT NULL UNIQUE REFERENCES model(id), tx real NOT NULL DEFAULT 0, ty real NOT NULL DEFAULT 0, tz real NOT NULL DEFAULT 0, rx real NOT NULL DEFAULT 0, ry real NOT NULL DEFAULT 0, rz real NOT NULL DEFAULT 0, sx real NOT NULL DEFAULT 1, sy real NOT NULL DEFAULT 1, sz real NOT NULL DEFAULT 1) STRICT`,
			`CREATE TABLE vertex (id integer primary key, model_id integer NOT NULL REFERENCES model(id), x real NOT NULL, y real NOT NULL, z real NOT NULL) STRICT`,
			`CREATE VIEW model_vertex AS SELECT v.id AS vertex_id, m.id AS model_id, ((cos(COALESCE(mt.ry,0))*cos(COALESCE(mt.rz,0))*(v.x*COALESCE(mt.sx, 1)) + (-cos(COALESCE(mt.ry,0))*sin(COALESCE(mt.rz,0)))*(v.y*COALESCE(mt.sy, 1)) + sin(COALESCE(mt.ry,0))*(v.z*COALESCE(mt.sz, 1)))-COALESCE(mt.tx, 0)) AS x, ((((cos(COALESCE(mt.rx,0))*sin(COALESCE(mt.rz,0)) + sin(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*cos(COALESCE(mt.rz,0)))*(v.x*COALESCE(mt.sx, 1)) + (cos(COALESCE(mt.rx,0))*cos(COALESCE(mt.rz,0)) - sin(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*sin(COALESCE(mt.rz,0)))*(v.y*COALESCE(mt.sy, 1)) + (-sin(COALESCE(mt.rx,0))*cos(COALESCE(mt.ry,0)))*(v.z*COALESCE(mt.sz, 1))))-COALESCE(mt.ty, 0)) AS y, (((sin(COALESCE(mt.rx,0))*sin(COALESCE(mt.rz,0)) - cos(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*cos(COALESCE(mt.rz,0)))*(v.x*COALESCE(mt.sx, 1)) + (sin(COALESCE(mt.rx,0))*cos(COALESCE(mt.rz,0)) + cos(COALESCE(mt.rx,0))*sin(COALESCE(mt.ry,0))*sin(COALESCE(mt.rz,0)))*(v.y*COALESCE(mt.sy, 1)) + (cos(COALESCE(mt.rx,0))*cos(COALESCE(mt.ry,0)))*(v.z*COALESCE(mt.sz, 1)))-COALESCE(mt.tz, 0)) AS z FROM vertex v INNER JOIN model m ON m.id = v.model_id LEFT JOIN model_transform mt ON mt.model_id = m.id`,
			`CREATE VIEW camera_vertex AS SELECT v.vertex_id AS vertex_id, c.id AS camera_id, (v.x-c.tx) AS x, (v.y-c.ty) AS y, (v.z-c.tz) AS z FROM model_vertex v, camera c`,
			`CREATE VIEW perspective_vertex AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/tan(c.fov/2)*c.aspect) AS x, (v.y/tan(c.fov/2)) AS y, (-v.z*((c.far+c.near)/(c.far-c.near))-((2*c.far*c.near)/(c.far-c.near))) AS z, -v.z as w FROM camera_vertex v INNER JOIN camera c ON c.id = v.camera_id`,
			`CREATE VIEW ndc_vertex AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/v.w) AS x, (v.y/v.w) AS y, (v.z/v.w) AS z, v.w AS w FROM perspective_vertex v`,

			`CREATE VIEW ndc_edge AS SELECT edge.id AS id, a.camera_id AS camera_id, (a.x/a.w) AS x1, (a.y/a.w) AS y1, (a.z/a.w) AS z1, (a.w/a.w) AS w1, (b.x/b.w) AS x2, (b.y/b.w) AS y2, (b.z/b.w) AS z2, b.w AS w2 FROM edge INNER JOIN perspective_vertex a ON a.vertex_id = edge.start_vertex_id INNER JOIN perspective_vertex b ON b.vertex_id = edge.end_vertex_id WHERE a.camera_id = b.camera_id`,
			//`CREATE VIEW ndc_face AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/v.w) AS x, (v.y/v.w) AS y, (v.z/v.w) AS z, v.w AS w FROM perspective_vertex v`,

			`CREATE VIEW projected_vertex AS SELECT vp.id AS viewport_id, v.vertex_id AS vertex_id, v.camera_id AS camera_id, vp.width * (0.5 + 0.5*v.x) AS x, vp.height * (0.5 + 0.5*v.y) AS y, v.z AS z, v.w AS w FROM ndc_vertex v INNER JOIN viewport vp ON vp.camera_id = v.camera_id WHERE abs(v.x) < 1 AND  abs(v.y) < 1  AND abs(v.z) < 1`,
			`CREATE VIEW projected_edge AS SELECT vp.id AS viewport_id, e.id AS edge_id, e.camera_id AS camera_id, vp.width * (0.5 + 0.5*e.x1) AS x1, vp.height * (0.5 + 0.5*e.y1) AS y1, e.z1 AS z1, e.w1 AS w1, vp.width * (0.5 + 0.5*e.x2) AS x2, vp.height * (0.5 + 0.5*e.y2) AS y2, e.z2 AS z2, e.w2 AS w2 FROM ndc_edge e INNER JOIN viewport vp ON vp.camera_id = e.camera_id WHERE abs(e.x1) < 1 AND  abs(e.y1) < 1  AND abs(e.z1) < 1 AND  abs(e.x2) < 1 AND  abs(e.y2) < 1  AND abs(e.z2) < 1`,
			`CREATE VIEW projected_face AS SELECT v.viewport_id AS viewport_id, IF(face_area.area_twice > 0, "blue", "red") AS clockcolor, face_area.area_twice > 0 AS clockwise, face.id AS face_id, v.vertex_id AS vertex_id, GROUP_CONCAT(CONCAT(v.x, ", ", v.y), ' ') AS points FROM face INNER JOIN face_projected_signed_area face_area ON face_area.id = face.id INNER JOIN face_vertex fv ON fv.face_id = face.id INNER JOIN projected_vertex v ON v.vertex_id = fv.vertex_id GROUP BY fv.face_id ORDER BY MAX(v.z) DESC`,
			`CREATE TABLE edge (id integer primary key, model_id integer NOT NULL REFERENCES model(id), start_vertex_id integer NOT NULL REFERENCES vertex(id), end_vertex_id integer NOT NULL REFERENCES vertex(id)) STRICT`,
			`CREATE TABLE face (id integer primary key, model_id integer NOT NULL REFERENCES model(id)) STRICT`,
			`CREATE TABLE face_vertex (id integer primary key, model_id integer NOT NULL REFERENCES model(id), face_id integer NOT NULL REFERENCES face(id), vertex_id integer NOT NULL REFERENCES vertex(id), sort integer) STRICT`,
			`CREATE VIEW face_vertex_neighbor AS SELECT face_vertex.*,
			         LEAD(face_vertex.id) OVER w AS next_vertex_id,
			         LAG(face_vertex.id) OVER w AS prev_vertex_id
			  FROM face_vertex
			  WINDOW w AS (PARTITION BY face_id ORDER BY sort)`,
			`CREATE VIEW face_vertex_neighbor_wrapped AS SELECT inner.id AS id, inner.face_id AS face_id, 
			         COALESCE(next_vertex_id, first_id) AS id_next,
			         COALESCE(prev_vertex_id, last_id) AS id_prev,
			next_vertex_id,
prev_vertex_id,
			first_id,
last_id
			  FROM (
			    SELECT fv.*,
					   fvn.next_vertex_id AS next_vertex_id,
					   fvn.prev_vertex_id AS prev_vertex_id,
			           FIRST_VALUE(fvn.id) OVER (PARTITION BY fvn.face_id ORDER BY fvn.sort) AS first_id,
			           FIRST_VALUE(fvn.id) OVER (PARTITION BY fvn.face_id ORDER BY -fvn.sort) AS last_id
			    FROM face_vertex_neighbor fvn 
			    INNER JOIN face_vertex fv ON fv.id = fvn.id
			  ) AS inner`,
			`CREATE VIEW face_projected_signed_area AS SELECT fvw.face_id AS id,
			 SUM((v1.x * v2.y) - (v2.x * v1.y)) AS area_twice
			FROM face_vertex_neighbor_wrapped fvw 
			INNER JOIN projected_vertex v1 ON v1.vertex_id = fvw.id
			INNER JOIN projected_vertex v2 ON v2.vertex_id = fvw.id_next
			GROUP BY fvw.face_id`,
			`INSERT INTO viewport(id, camera_id, width,height) VALUES(1, 1, 1, 1)`,
			`INSERT INTO camera(id,tx,ty,tz, near, far, fov, aspect) VALUES(1, 0, 0, 50, 0.1, 100, 1, 1)`,
			`INSERT INTO model(id) VALUES(1)`,
			`INSERT INTO model_transform(id, model_id) VALUES(1, 1)`,
			`INSERT INTO vertex(model_id, id, x, y, z) VALUES(1, 1, 1, 2, 3), (1, 2, 3, 2, 3), (1, 3, -1, -3, 3),(1, 4, -1, 4, 5), (1, 5, -3, 2, 6), (1, 6, 3, -3, 7)`,
			`INSERT INTO edge(model_id, id, start_vertex_id, end_vertex_id) VALUES (1, 1, 1, 2), (1, 2, 2, 3), (1, 3, 3, 1), (1, 4, 4, 5), (1, 5, 5, 6), (1, 6, 6, 4)`,
			`INSERT INTO face(model_id, id) VALUES(1, 1)`,
			`INSERT INTO face(model_id, id) VALUES(1, 2)`,
			`INSERT INTO face_vertex(model_id, face_id, id, vertex_id, sort) VALUES(1, 1, 1, 1, 1), (1, 1, 2, 2, 2), (1, 1, 3, 3, 3), (1, 2, 4, 4, 1), (1, 2, 5, 5, 2), (1, 2, 6, 6, 3)`,
		].join(";\n");
		db.run(sqlstr); // Run the query without returning anything

		statements = {
			cameraSelect: db.prepare("SELECT * FROM camera"),
			transformSelect: db.prepare("SELECT * FROM model_transform"),
			viewportSelect: db.prepare("SELECT * FROM viewport"),
			vertexSelect: db.prepare(
				"SELECT vertex_id AS id, x, y, z, w FROM projected_vertex WHERE viewport_id = $viewport",
			),
			edgeSelect: db.prepare(
				"SELECT * FROM projected_edge WHERE viewport_id = $viewport",
			),
			faceSelect: db.prepare(
				"SELECT * FROM projected_face WHERE viewport_id = $viewport AND ($clockwise = clockwise) IS DISTINCT FROM FALSE",
			),
		};
		setViewport(db, window.innerWidth, window.innerHeight);

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

	function setTransform(db, { tx, ty, tz, rx, ry, rz, sx, sy, sz }) {
		db.run(
			"UPDATE model_transform SET tx=?, ty=?, tz=?, rx=?, ry=?, rz=?, sx=?, sy=?, sz=? WHERE id = 1",
			[tx, ty, tz, rx, ry, rz, sx, sy, sz],
		);
		update(db);
	}

	function setViewport(db, w, h) {
		db.run("UPDATE viewport SET width=?,height=? WHERE id = 1", [w, h]);
		update(db);
	}

	function update(db) {
		if (db) {
			const test = db.prepare(
				"SELECT id, id_next FROM face_vertex_neighbor_wrapped",
			);
			while (test.step()) {
				console.log(test.getAsObject());
			}
			{
				if (statements.cameraSelect.step()) {
					camera = statements.cameraSelect.getAsObject();
				}
			}
			{
				if (statements.transformSelect.step()) {
					transform = statements.transformSelect.getAsObject();
				}
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
	<div class="overlay">
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

		<form
			oninput={(evt) => {
				setTransform(
					db,
					Object.fromEntries(new FormData(evt.currentTarget)),
				);
			}}
		>
			<div class="slider-group">
				<label
					><span>tx:</span>
					<input
						name="tx"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.tx}
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
						value={transform.ty}
					/></label
				><label
					><span>tz:</span>
					<input
						name="tz"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.tz}
					/></label
				>
				<label
					><span>rx:</span>
					<input
						name="rx"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.rx}
					/></label
				>
				<label
					><span>ry:</span>
					<input
						name="ry"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.ry}
					/></label
				><label
					><span>rz:</span>
					<input
						name="rz"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.rz}
					/></label
				>

				<label
					><span>sx:</span>
					<input
						name="sx"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.sx}
					/></label
				>
				<label
					><span>sy:</span>
					<input
						name="sy"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.sy}
					/></label
				><label
					><span>sz:</span>
					<input
						name="sz"
						step="0.0001"
						type="range"
						min="-60"
						max="60"
						value={transform.sz}
					/></label
				>
			</div>
		</form>

		<div class="debug">
			<div>
				{JSON.stringify(vertices, null, "  ")}
			</div>
			<div>
				{JSON.stringify(camera, null, "  ")}
			</div>
		</div>
	</div>

	<svg class="viewport" viewBox="0 0 {viewport.width} {viewport.height}">
		{#each faces as f}
			<polygon
				fill={f.clockcolor}
				stroke={"black"}
				stroke-width="5"
				points={f.points}
			></polygon>
		{/each}

		{#each edges as e}
			<line
				stroke-width="1"
				stroke="black"
				r="0.1"
				x1={e.x1}
				y1={e.y1}
				x2={e.x2}
				y2={e.y2}
			></line>
		{/each}
		{#each vertices as v}
			<circle r="5" cx={v.x} cy={v.y}></circle>
		{/each}
	</svg>
</div>

<style>
	:global(body) {
		margin: 0;
	}
	svg {
		position: absolute;
		left: 0;
		right: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}
	.app {
		display: grid;
		grid-template-columns: 1fr 1fr 1fr;
		grid-template-rows: 1fr 1fr 1fr;
		position: absolute;
		width: 100vw;
		height: 100vh;
	}

	.overlay {
		grid-area: 1/1 / 4 /1;
		background: #aaa3;
		padding: 1em;
		margin: 1em;
		gap: 1em;
		display: grid;
		align-content: start;
	}

	.debug {
		overflow: auto;
		max-height: 20em;
		background: #fff;
		padding: 1ex;
		white-space: pre;
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
</style>
