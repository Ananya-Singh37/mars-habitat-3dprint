# mars-habitat-3dprint
Parametric 3D-printable parts for a Mars Habitat outfitting challenge — hatch rings, pipe clamps, shelf brackets, and storage bins generated with Python + OpenSCAD

#!/usr/bin/env python3
"""
generate_mars_parts.py

Generates parametric OpenSCAD files for a Mars-hab 3D-print challenge:
 - hatch_ring.scad
 - pipe_clamp.scad
 - shelf_bracket.scad
 - storage_bin.scad

Open each file in OpenSCAD to preview and export STL.
Author: ChatGPT (assistant)
"""

import math
from pathlib import Path

OUT_DIR = Path(".")
OUT_DIR.mkdir(exist_ok=True)

# Helper: write a .scad file with a header describing parameters
def write_scad(filename: str, content: str):
    p = OUT_DIR / filename
    p.write_text(content)
    print(f"Wrote {p.resolve()}")


### 1) Hatch Ring (bolted interface)
hatch_ring = """// hatch_ring.scad
// Parametric hatch ring for a small airlock or hatch.
// Parameters: outer_diam, inner_diam, thickness, bolt_count, bolt_hole_d
// Use OpenSCAD to render and export STL.

outer_diam = 260;       // mm -- outer diameter of ring
inner_diam = 200;       // mm -- inner opening diameter
thickness = 8;          // mm -- ring thickness (height)
bolt_count = 8;
bolt_hole_d = 6;        // mm

$fn = 128;

// Ring body
difference() {
    linear_extrude(height = thickness)
        difference() {
            circle(d = outer_diam);
            translate([0,0,0]) circle(d = inner_diam);
        }
}

// Bolt holes
for (i = [0:bolt_count-1]) {
    angle = 360/bolt_count * i;
    r = (inner_diam + outer_diam)/4; // radial position for bolt holes
    x = r * cos(angle);
    y = r * sin(angle);
    translate([x, y, -1]) rotate([0,0,0]) cylinder(h = thickness + 2, d = bolt_hole_d, $fn = 48);
}

// Optional countersink (simple chamfer)
module countersink(diam, depth, cnt_x, cnt_y) {
    // simple cone cutter
    translate([cnt_x, cnt_y, -2])
        rotate([0,180,0]) cone(h=depth+2, r1=diam/2*1.8, r2=diam/2);
}

// note: OpenSCAD doesn't have a native cone - implement via linear_extrude of scaled circle
module cone(h, r1, r2) {
    // approximate cone by scaling a circle while extruding a polygon
    translate([0,0,0]) linear_extrude(height=h, scale = r2/r1)
        circle(d = r1*2, $fn = 48);
}
"""

write_scad("hatch_ring.scad", hatch_ring)


### 2) Pipe Clamp (split clamp with screw boss)
pipe_clamp = """// pipe_clamp.scad
// Parametric split clamp for piping or conduit.
// Parameters: pipe_d, wall_thickness, clamp_width, screw_d, screw_clearance_d

pipe_d = 25;            // mm internal pipe diameter to clamp around
wall_thickness = 4;     // mm thickness of clamp material
clamp_width = 30;       // mm axial width of clamp
screw_clearance_d = 5;  // hole clearance for M4 ~ 4.2 mm, use 5 mm to be safe
gap = 2.5;              // mm split gap to allow tightening

$fn = 64;

module clamp_half() {
    difference() {
        // outer shape
        hull() {
            translate([0, pipe_d/2 + wall_thickness, 0]) circle(d = wall_thickness*2);
            translate([clamp_width, pipe_d/2 + wall_thickness, 0]) circle(d = wall_thickness*2);
        }
        // pipe cutout
        translate([clamp_width/2, pipe_d/2, 0])
            rotate([0,0,0]) circle(d = pipe_d + 0.5, $fn = 128);
        // inner clearance bottom
        translate([clamp_width/2, pipe_d/2 - 6, -1]) square([clamp_width + 4, 12], center = true);
    }
}

difference() {
    union() {
        translate([0,0,0]) linear_extrude(height = wall_thickness) clamp_half();
        translate([0,0,wall_thickness]) mirror([1,0,0]) linear_extrude(height = wall_thickness) clamp_half();
    }
    // split gap
    translate([clamp_width/2, pipe_d/2, -2]) cube([gap, pipe_d + 20, wall_thickness + 4], center = true);
    // screw holes
    translate([clamp_width*0.15, pipe_d/2 + wall_thickness*0.6, -2]) cylinder(h = wall_thickness + 6, d = screw_clearance_d, $fn = 36);
    translate([clamp_width*0.85, pipe_d/2 + wall_thickness*0.6, -2]) cylinder(h = wall_thickness + 6, d = screw_clearance_d, $fn = 36);
}
"""

write_scad("pipe_clamp.scad", pipe_clamp)


### 3) Shelf Bracket (snap-fit)
shelf_bracket = """// shelf_bracket.scad
// Parametric shelf bracket for interior hab mounting (snap-fit to slotted rail).
// Parameters: bracket_length, bracket_height, tab_thickness, clip_depth

bracket_length = 120;
bracket_height = 80;
tab_thickness = 4;
clip_depth = 10;

$fn = 32;

module bracket() {
    // L-shaped bracket
    union() {
        // vertical
        translate([0,0,0]) cube([tab_thickness, bracket_height, 12], center = false);
        // horizontal shelf support
        translate([tab_thickness, bracket_height - 12, 0]) cube([bracket_length, 12, 12], center = false);
        // diagonal rib
        translate([tab_thickness+10, bracket_height/2, -1]) rotate([0,0, -30]) cube([bracket_length/3, 10, 14], center = false);
    }
    // snap clip on vertical
    translate([-1, bracket_height - clip_depth - 2, 0]) cube([6, clip_depth + 4, 6], center = false);
}

difference() {
    bracket();
    // cutout to reduce material (lightweight)
    translate([tab_thickness + 6, 12, -2]) cube([bracket_length - 20, bracket_height - 30, 16], center = false);
}
bracket();
"""

write_scad("shelf_bracket.scad", shelf_bracket)


### 4) Modular Storage Bin (stackable)
storage_bin = """// storage_bin.scad
// Modular stackable storage bin. Dovetail on top/bottom to lock when stacked.
// Parameters: length, width, height, wall_thickness, dovetail_size

length = 120;
width = 80;
height = 60;
wall_thickness = 3;
dovetail_size = 6;

$fn = 32;

module bin_body() {
    difference() {
        // outer
        translate([0,0,0]) cube([length, width, height], center = false);
        // inner cavity
        translate([wall_thickness, wall_thickness, wall_thickness]) cube(
            [length - 2*wall_thickness, width - 2*wall_thickness, height - wall_thickness], center = false);
    }
}

module dovetail_top() {
    translate([length*0.05, width/2 - dovetail_size*1.5, height - 1])
        linear_extrude(height = 2) polygon(points=[
            [0,0],
            [dovetail_size*3, 0],
            [dovetail_size*2, dovetail_size],
            [dovetail_size, dovetail_size]
        ]);
}

module dovetail_bottom() {
    translate([length*0.05, width/2 - dovetail_size*1.5, -2])
        linear_extrude(height = 2) polygon(points=[
            [0,dovetail_size],
            [dovetail_size*3, dovetail_size],
            [dovetail_size*2, 0],
            [dovetail_size, 0]
        ]);
}

difference() {
    union() {
        bin_body();
        dovetail_top();
        dovetail_bottom();
    }
    // optional handle cutout
    translate([length*0.5 - 20, -2, height*0.5 - 8]) cube([40, 6, 16], center = false);
}

"""

write_scad("storage_bin.scad", storage_bin)


### 5) Challenge README (instructions and scoring)
readme = """Mars Habitat 3D-Print Challenge — Generated Parts
----------------------------------------------------

Files generated:
 - hatch_ring.scad
 - pipe_clamp.scad
 - shelf_bracket.scad
 - storage_bin.scad

How to use:
 1. Open a .scad file in OpenSCAD (https://openscad.org).
 2. Adjust parameters at the top of each file (dimensions in mm).
 3. Press F5 to preview, F6 to render, then 'Design -> Export as STL'.

Suggested challenge tasks:
 - Optimize the Hatch Ring to minimize mass while maintaining a 6 mm solid flange for bolting.
 - Make the Pipe Clamp support a 25 mm pipe with minimal material but < 2 mm flex under 5 N lateral load (simulation by printing and testing).
 - Make the Shelf Bracket snap into a simulated rail (create a matching rail) and support 2 kg with one bracket.
 - Optimize the Storage Bin to stack securely with dovetail, maximize internal volume, and ensure corners have 3 mm fillets.

Scoring (example):
 - Mass Efficiency (30 pts): lowest mass wins.
 - Strength (30 pts): passes a 2 kg load test without failure.
 - Printability (20 pts): minimal supports, prints under 6 hours on standard FDM.
 - Creativity / Integration (20 pts): clever features for Mars-hab constraints (thermal retention, tether points, redundancy).

Notes:
 - For pressure-loaded components (hatches), real engineering review and certification is required — these models are prototypes or mockups for practice only.
 - Use materials suitable for your printers: PLA is fine for prototypes; PETG/ABS/ASA for improved heat and chemical resistance.

Good luck — happy printing on (simulated) Martian soil! 🚀
"""

write_scad("README_MARS_3D_CHALLENGE.txt", readme)

print("All files generated. Open the .scad files in OpenSCAD to view and export STLs.")

 
