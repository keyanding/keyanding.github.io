---
layout: post
title: Getting cross-track error right on WGS-84
date: 2026-11-02 08:00:00 +0800
author: Keyan Ding
description: A spherical estimate, a missing sign, and what 1,320 test cases taught me about turning geometry into code.
categories: ["Engineering Notes"]
related_posts: false
_styles: |
  .geodesic-article .standfirst { font-size: 1.2rem; line-height: 1.6; color: var(--global-text-color-light); }
  .geodesic-article h2 { margin-top: 2.3rem; }
  .geodesic-article .equation { text-align: center; font-size: 1.25rem; margin: 1.5rem 0; }
  .geodesic-article figure { margin: 2rem 0; }
  .geodesic-article figure a { display: block; overflow-x: auto; }
  .geodesic-article figure img { display: block; width: 100%; height: auto; background: white; }
  .geodesic-article figcaption { font-size: 0.85rem; line-height: 1.5; color: var(--global-text-color-light); margin-top: 0.75rem; }
  .geodesic-article table { width: 100%; border-collapse: collapse; font-size: 0.9rem; margin: 1.5rem 0; }
  .geodesic-article th, .geodesic-article td { padding: 0.65rem; border-bottom: 1px solid var(--global-divider-color); vertical-align: top; text-align: left; }
  .geodesic-article th { background: var(--global-code-bg-color); }
  @media (max-width: 600px) { .geodesic-article figure img { min-width: 760px; } }
---

<div class="geodesic-article">
<p class="standfirst">A spherical estimate, a missing sign, and what 1,320 test cases taught me about turning geometry into code.</p>

<p>Given two waypoints and an aircraft position, I wanted the signed distance from the aircraft to the geodesic through those waypoints. The projection could fall before the start or beyond the end. I chose positive for the left side of the route and negative for the right.</p>
<p>The geometry fits on a page. The implementation needed much more care about direction, stopping conditions, and what an error measurement actually means.</p>

<h2>A sphere makes a useful first estimate</h2>
<p>On a sphere, the route lies in a plane through the Earth's center. A cross product gives the plane's normal; projecting the aircraft's unit vector onto that plane and normalizing it gives a nearby point on the great circle.</p>
<p>Earth is flattened at the poles. <a href="https://earth-info.nga.mil/index.php?action=wgs84&amp;dir=wgs84">WGS-84</a> includes an ellipsoidal Earth model, so a spherical projection generally differs from the ellipsoidal one. For this implementation I used the direct and inverse geodesic formulas in Appendix E of <a href="https://store.accuristech.com/standards/rtca-do-283c?product_id=3041021">RTCA DO-283C</a>, the minimum operational performance standard for RNP area-navigation equipment. The spherical result became the initial estimate.</p>

<h2>The solver</h2>
<p>Let <var>Q(s)</var> be the point reached by traveling signed distance <var>s</var> from the start along the route. Negative distances extend behind the start. At <var>Q(s)</var>, let <var>α</var> be the route's forward tangent bearing and <var>β</var> the bearing toward the aircraft. I solve the local perpendicularity condition:</p>
<div class="equation"><var>f(s)</var> = cosd(<var>β(s)</var> − <var>α(s)</var>) = 0</div>
<p>Here, <code>cosd</code> takes degrees. A point coincident with the aircraft is handled directly because its bearing to the aircraft is undefined.</p>
<ol>
<li>Compute the spherical estimate <var>P₀</var>, then convert its distance from the start into a signed initial distance <var>s₀</var>.</li>
<li>Evaluate the actual ellipsoidal point <var>Q(s₀)</var>. Accept it if the residual magnitude is at most <code>1e-4</code>.</li>
<li>Otherwise, start at <var>s₀</var> ± 185 m. Use secant interpolation, replace the endpoint with the same residual sign, and use the midpoint if the denominator is zero or interpolation leaves the interval.</li>
<li>Stop on the residual tolerance or a width of 0.001 m. After 20 iterations, return the remaining interval's midpoint and record that fallback separately.</li>
</ol>
<p>This is a false-position update when the endpoints bracket a root. The final version uses a fixed interval without expansion.</p>

<h2>An unsigned initial distance</h2>
<p>An inverse geodesic returns a nonnegative distance. My along-route coordinate needed a sign. Without it, a projection 100 km behind the start becomes an estimate 100 km ahead.</p>
<p>I determine the sign from the angle between the start-to-<var>P₀</var> bearing and the route's initial bearing. If their cosine is negative, I negate the distance. This selects the backward direction for the local initial estimate.</p>
<p>To isolate the effect, I reran test case 820 with only that correction disabled. The route runs east along the equator for 100 km. The aircraft is 10 km south of a reference foot 100 km before the start.</p>
<figure class="wide"><a href="{{ '/assets/img/posts/wgs84-cross-track/signed-seed-comparison.svg' | relative_url }}"><img src="{{ '/assets/img/posts/wgs84-cross-track/signed-seed-comparison.svg' | relative_url }}" alt="Side-by-side controlled test. Without the seed sign, the returned foot is 100.185 km ahead of the start and 200.185 km from the reference. With the sign restored, the foot is correctly 100 km behind the start."></a><figcaption>Equatorial-route schematic from the MATLAB results. The left panel disables one correction in the final solver; it is not a replay of every earlier implementation.</figcaption></figure>
<p>The unsigned interval never contained the foot. It still became narrow enough to stop, returning a cross-track distance of −200.435 km instead of −10 km. Restoring the sign let the initial residual check accept the correct foot without iteration. A small interval is useful evidence only when it encloses the intended solution.</p>

<h2>Other details that changed the result</h2>
<p>For the left/right sign, I initially considered the bearing from the projection to the destination waypoint. Beyond the destination, that bearing points backward; at the destination itself, it is undefined. Reusing the forward tangent bearing from the direct geodesic calculation fixes both cases. I also use one angle convention throughout: [−180°, 180°).</p>
<p>The tolerances have different units. A residual of <code>1e-4</code> is dimensionless; an interval width of 0.001 m is a distance. In one test, the initial residual passed even though the foot was 1.831 m from the reference. Its cross-track distance error was only about 0.091 mm. Accepting a small residual does not imply millimeter accuracy in the projected coordinates.</p>

<h2>What I tested</h2>
<p>The suite contains 810 combinations of six routes, 15 along-route positions, and nine lateral offsets, plus ten fixed cases and 500 seeded random cases. The combinations place the foot before the start, inside the segment, and beyond the end.</p>
<div class="table-responsive"><table><thead><tr><th>Factor</th><th>Coverage</th></tr></thead><tbody>
<tr><td>Start latitude</td><td>About −79.86° to +85°; both hemispheres and the equator</td></tr>
<tr><td>Route length</td><td>1 m to 8,000 km</td></tr>
<tr><td>Distance outside either endpoint</td><td>0.01 m, 100 m, and 1, 10, 100, 500 NM in the combination grid</td></tr>
<tr><td>Lateral offset</td><td>0, ±0.01 m, ±1 m, ±1 NM, ±10 NM in the grid; no case exceeds 10 NM</td></tr>
<tr><td>Direction and longitude</td><td>Northbound, eastbound, oblique routes, and longitude-wrap crossings</td></tr>
</tbody></table></div>
<p>The random offsets were clipped to ±10 NM: 476 of the 500 landed on those boundaries. This part of the suite stresses the limit rather than sampling the interior uniformly.</p>
<figure class="wide"><a href="{{ '/assets/img/posts/wgs84-cross-track/iteration-results.svg' | relative_url }}"><img src="{{ '/assets/img/posts/wgs84-cross-track/iteration-results.svg' | relative_url }}" alt="Histogram of iterations per case and exit-condition counts. Most cases require zero or one iteration. Endpoint coincidence ends 13 cases, initial residual 354, iterated residual 826, interval width 127, and iteration limit zero."></a><figcaption>All 45 cases reaching iteration 20 passed its width check. None used the iteration-limit fallback.</figcaption></figure>
<div class="table-responsive"><table><thead><tr><th>Latest result</th><th>Measured value</th></tr></thead><tbody>
<tr><td>Cases passing the test criteria</td><td>1,320 / 1,320</td></tr>
<tr><td>Initially opposite signs, among cases entering the loop</td><td>953 / 953</td></tr>
<tr><td>Opposite signs in recorded iteration intervals</td><td>3,119 / 3,119</td></tr>
<tr><td>Mean / maximum iterations</td><td>2.363 / 20, including zero-iteration cases</td></tr>
<tr><td>Maximum cross-track distance error</td><td>0.353 mm</td></tr>
<tr><td>Maximum projection-position error</td><td>1.831 m, using the test's local coordinate-distance metric</td></tr>
</tbody></table></div>
<p>Passing meant distance error below 2 mm, position error below 2 m, correct signs for nonzero offsets, and at most 20 iterations. The reference geometry shares the direct solver, so these are consistency tests. Seven standard reference cases and 30 independent geodesic ODE comparisons provide separate checks on the geodesic routines.</p>

<h2>What this experiment taught me</h2>
<p>I now treat the operating envelope as part of the algorithm: different latitudes, longitude crossings, and aircraft positions whose projections fall before, within, or beyond the route segment. A useful test suite also needs points on the route and extremely close to it. Ordinary mid-segment examples would have missed several of the problems here.</p>
<p>The secant formula is simple, but implementing it still means checking the conditions around it. For this interval-based variant, I verified that the initial endpoints had opposite signs, endpoint updates preserved that sign change, and successive intervals became smaller. These checks do not guarantee rapid convergence: one endpoint can remain fixed while progress slows. Convergence speed therefore needs its own checks against the iteration budget.</p>
<p>The objective function needs attention too. Near the route, the bearing toward the aircraft can change sharply as the candidate foot passes it. On the route itself, that bearing is undefined at coincidence, and the cosine residual can jump between opposite signs rather than smoothly reach zero. A residual-only stopping rule is therefore insufficient: coincidence handling and a justified positional stopping condition matter too. The secant denominator also needs its own zero check.</p>
<p>I would make angle ranges, bearing directions, signed distances, and units explicit at every interface. I would also choose tolerances from the required output accuracy, then measure function evaluations and runtime: tighter projection coordinates may cost more without materially improving cross-track distance. The 20-iteration limit bounds the outer loop, not total execution time or error. Every exit should say whether it met a tolerance or merely exhausted its budget. These tests support the cases measured; they do not prove convergence everywhere.</p>
</div>
