<script>
	import { onMount } from "svelte";
	import initSqlJs from "sql.js";
	// or if you are in a browser:
	// const initSqlJs = window.initSqlJs;

	let vertices = $state(undefined);
	let edges = $state(undefined);
	let faces = $state(undefined);
	let camera = $state({});

	let db;

	onMount(async () => {
		const SQL = await initSqlJs({
			// Required to load the wasm binary asynchronously. Of course, you can host it wherever you want
			// You can omit locateFile completely when running in node
			locateFile: (file) => `https://sql.js.org/dist/${file}`,
		});

		db = new SQL.Database();
		let sqlstr = [
			"CREATE TABLE camera (id integer primary key, tx real NOT NULL, ty real NOT NULL, tz real NOT NULL, near real NOT NULL, far real NOT NULL, fov real NOT NULL, aspect real NOT NULL) STRICT",
			"CREATE TABLE viewport (id integer primary key, width REAL NOT NULL, height REAL NOT NULL) STRICT",
			"CREATE TABLE model (id integer primary key) STRICT",
			"CREATE TABLE vertex (id integer primary key, model_id integer NOT NULL REFERENCES model(id), x real NOT NULL, y real NOT NULL, z real NOT NULL) STRICT",
			"CREATE VIEW camera_vertex AS SELECT v.id AS vertex_id, c.id AS camera_id, (v.x-c.tx) AS x, (v.y-c.ty) AS y, (v.z-c.tz) AS z FROM vertex v, camera c",
			"CREATE VIEW perspective_vertex AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/tan(c.fov/2)*c.aspect) AS x, (v.y/tan(c.fov/2)) AS y, (-v.z*((c.far+c.near)/(c.far-c.near))-((2*c.far*c.near)/(c.far-c.near))) AS z, -v.z as w FROM camera_vertex v INNER JOIN camera c ON c.id = v.camera_id",
			"CREATE VIEW projected_vertex AS SELECT v.vertex_id AS vertex_id, v.camera_id AS camera_id, (v.x/v.w) AS x, (v.y/v.w) AS y, (v.z/v.w) AS z, v.w AS w, v.z AS oz FROM perspective_vertex v WHERE abs(v.x) < abs(v.w) AND  abs(v.y) < abs(v.w)  AND v.z < v.w AND v.z > -v.w",
			"CREATE TABLE edge (id integer primary key, model_id integer NOT NULL REFERENCES model(id), start_vertex_id integer NOT NULL REFERENCES vertex(id), end_vertex_id integer NOT NULL REFERENCES vertex(id)) STRICT",
			"CREATE TABLE face (id integer primary key, model_id integer NOT NULL REFERENCES model(id)) STRICT",
			"CREATE TABLE face_vertex (id integer primary key, model_id integer NOT NULL REFERENCES model(id), face_id integer NOT NULL REFERENCES face(id), vertex_id integer NOT NULL REFERENCES vertex(id), sort integer) STRICT",
			"INSERT INTO viewport(id,width,height) VALUES(1, 1, 1)",
			"INSERT INTO camera(id,tx,ty,tz, near, far, fov, aspect) VALUES(1, 0, 0, 0.1, 3, 100, 1, 1)",
			"INSERT INTO model(id) VALUES(1)",
			"INSERT INTO vertex(model_id, id, x, y, z) VALUES(1, 1, 1, 2, 3), (1, 2, 3, 2, 3), (1, 3, -1, -3, 3),(1, 4, -1, 4, 5), (1, 5, -3, 2, 6), (1, 6, 3, -3, 7)",
			"INSERT INTO edge(model_id, id, start_vertex_id, end_vertex_id) VALUES(1, 1, 1, 2), (1, 2, 2, 3), (1, 3, 3, 1)",
			"INSERT INTO face(model_id, id) VALUES(1, 1)",
			"INSERT INTO face(model_id, id) VALUES(1, 2)",
			"INSERT INTO face_vertex(model_id, face_id, id, vertex_id, sort) VALUES(1, 1, 1, 1, 1), (1, 1, 2, 2, 2), (1, 1, 3, 3, 3), (1, 2, 4, 4, 5), (1, 2, 5, 5, 6), (1, 2, 6, 6, 4)",
		].join(";\n");
		db.run(sqlstr); // Run the query without returning anything

		update(db);

		setViewport(db, window.innerWidth, window.innerHeight);

		return () => {
			vertices = undefined;
			edges = undefined;
			faces = undefined;
		};
	});

	function setCamera(db, x, y, z, near, far, fov) {
		db.run(
			"UPDATE camera SET tx=?,ty=?,tz=?,near=?,far=?, fov=? WHERE id = 1",
			[x, y, z, near, far, fov],
		);
		update(db);
	}

	function setViewport(db, w, h) {
		db.run("UPDATE viewport SET width=?,height=? WHERE id = 1", [w, h]);
		update(db);
	}

	function update(db) {
		if (db) {
			{
				const stmt = db.prepare("SELECT *  FROM camera");
				if (stmt.step()) {
					camera = stmt.getAsObject();
				}
			}

			// Prepare an sql statement
			{
				const stmt = db.prepare(
					"SELECT vertex_id AS id, x, y, z, w, oz FROM projected_vertex WHERE camera_id = 1",
				);
				const rows = [];
				while (stmt.step()) {
					//
					rows.push(stmt.getAsObject());
				}

				vertices = rows;
			}

			{
				const stmt = db.prepare(
					"SELECT edge.id AS id, a.x AS x1, a.y AS y1, a.z AS z1, b.x AS x2, b.y AS y2, b.z AS z2 FROM edge INNER JOIN projected_vertex a ON a.vertex_id = edge.start_vertex_id INNER JOIN projected_vertex b ON b.vertex_id = edge.end_vertex_id WHERE (a.camera_id, b.camera_id) = (1,1)",
				);
				const rows = [];
				while (stmt.step()) {
					//
					rows.push(stmt.getAsObject());
				}

				edges = rows;
			}

			{
				const stmt = db.prepare(
					"SELECT face.id AS face_id, v.vertex_id AS vertex_id, v.x AS x,v.y AS y, v.z AS z FROM face INNER JOIN face_vertex fv ON fv.face_id = face.id INNER JOIN projected_vertex v ON v.vertex_id = fv.vertex_id WHERE v.camera_id = 1 ORDER BY fv.sort ASC",
				);
				const rows = {};
				while (stmt.step()) {
					const face = stmt.getAsObject();
					if (!rows[face.face_id]) {
						rows[face.face_id] = [];
					}
					rows[face.face_id].push(face);
				}

				faces = Object.values(rows);
			}
		}
	}
</script>

<form
	oninput={(evt) => {
		const pos = Object.fromEntries(new FormData(evt.currentTarget));
		setCamera(db, pos.x, pos.y, pos.z, pos.near, pos.far, pos.fov);
	}}
>
	<input
		name="x"
		step="0.01"
		type="range"
		min="-60"
		max="60"
		value={camera.tx}
	/>
	<input
		name="y"
		step="0.01"
		type="range"
		min="-60"
		max="60"
		value={camera.ty}
	/>
	<input
		name="z"
		step="0.01"
		type="range"
		min="-60"
		max="60"
		value={camera.tz}
	/>
	<input
		name="near"
		step="0.01"
		type="range"
		min="0.1"
		max="100"
		value={camera.near}
	/>
	<input
		name="far"
		step="0.01"
		type="range"
		min="0.1"
		max="100"
		value={camera.far}
	/>
	<input
		name="fov"
		step="0.01"
		type="range"
		min="0.1"
		max="3"
		value={camera.fov}
	/>
</form>

<pre style="position: absolute;">{JSON.stringify(vertices, null, "  ")}
<br />
{JSON.stringify(camera, null, "  ")}</pre>

<svg viewBox="-1 -1 2 2">
	{#each faces as f}
		<polygon
			fill="#fee"
			points={f.map(({ x, y }) => `${x},${y}`).join(", ")}
		></polygon>
	{/each}

	{#each edges as e}
		<line
			stroke-width="0.01"
			stroke="black"
			r="0.1"
			x1={e.x1}
			y1={e.y1}
			x2={e.x2}
			y2={e.y2}
		></line>
	{/each}
	{#each vertices as v}
		<circle r="0.03" cx={v.x} cy={v.y}></circle>
	{/each}
</svg>

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
</style>
