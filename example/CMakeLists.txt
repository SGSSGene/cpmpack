# SPDX-FileCopyrightText: 2024 Simon Gene Gottlieb
# SPDX-License-Identifier: CC0-1.0
cmake_minimum_required (VERSION 3.31)

project(demo LANGUAGES CXX DESCRIPTION "demonstration of cpmpack")

include(cmake/CPM.cmake)
CPMAddPackage("gh:SGSSGene/cpmpack#main")
loadCPMPack("${CMAKE_CURRENT_SOURCE_DIR}/cpmpack.json")

