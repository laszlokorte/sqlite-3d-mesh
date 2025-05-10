# SQLite 3D Renderer

This is a demo of projecting 3d triangles via SQL queries.

Think of <code>SELECT project(...) as x, project(...) as y FROM model, vertex, camera, transform WHERE clockwise AND clipped IN BETWEEN -1 AND 1</code>