---
id: inspect-a-single-blueprint-graph-node-by-guid
title: Inspect a single Blueprint graph node by GUID
status: request
version: 26.606.148
tags: []
---

# Inspect a single Blueprint graph node by GUID

## What

A way to retrieve the full detail of **one** Blueprint graph node — identified by its node GUID — without returning the entire graph. Either a dedicated tool (e.g. `unreal_bp_node_info`) or an honored `node_guid` filter on `unreal_bp_graph_nodes_list`.

## Goal

Let an author read a single node's pins (names, directions, categories, sub-objects, default values) and links so they can wire it, in graphs of any size.

## Behaviour

- Input: `bp_path`, `graph_name`, `node_guid`.
- Output: that node only — `node_class`, member/function name, position, and the full pin list with per-pin direction, category, sub-category object, default value, and linked count.
- When `node_guid` is supplied to a listing tool, return only the matching node; do not fall back to the whole graph.
- Accept a short GUID prefix or require the full GUID — state which; if prefix, error clearly on ambiguity.
- Error clearly when the GUID is not found in the named graph.

## Why it matters

`unreal_bp_graph_nodes_list` currently returns every node even when a `node_guid` is passed. In large event graphs (600+ nodes) the response exceeds the tool output limit and must be dumped to a file and parsed externally just to read one node's pins. Single-node inspection is the common case during wiring.

## Acceptance

- Passing a valid `node_guid` returns exactly one node's detail, small enough to consume directly, regardless of total graph size.
- The returned pin names/directions match what `unreal_bp_node_link_pins` expects, so an author can wire from this output without a full-graph dump.
- An invalid/unknown GUID returns a clear not-found error, not the whole graph.
