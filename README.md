# CPMpack

Short declaration of libraries/executable and dependencies using CPM in the background.

## How to integrate

Must use [CPM.cmake](https://github.com/cpm-cmake/CPM.cmake).

```cmake
include(cmake/CPM.cmake)
CPMAddPackage("gh:SGSSGene/cpmpack@1.0.1")

# Loads and adds all packages from the JSON file
loadCPMPack("${CMAKE_CURRENT_SOURCE_DIR}/cpmpack.json")
```

## JSON file layout

An example layout:
```
{
    "format_version": "1",
    "options": [
      {
        "name": "MYLIB_BUILD_TEST",
        "description": "build tests",
        "default": "${PROJECT_IS_TOP_LEVEL}"
      }
    ],
    "translationsets": [
      {
          "name": "myLib::myLib",
          "type": "library",
          "language": "cxx_std_20"
      },
      {
           "if": "MYLIB_BUILD_TEST",
           "name": "myApp",
           "type": "executable",
           "language": "cxx_std_20",
           "dependencies": [
             "myLib::myLib"
           ]
      }
    ],
    "packages": [
        {
            "name": "nlohmann_json",
            "version": "3.11.3",
            "github_repository": "nlohmann/json",
            "options": [
                "json_buildtests off"
            ]
        },
        {
            "if": "MYLIB_BUILD_TEST",
            "name": "fmt",
            "version": "10.2.1",
            "git_tag": "{VERSION}",
            "github_repository": "fmtlib/fmt"
        }
    ],
}
```
`format_version`: Must be value "1", allows for later backwards compatibility.
`options`: A list of options the user can change
`translationsets`: A list of translation sets that can be compiled
`packages`: A list of objects. Each objects represents a `CPMAddPackage`/`CPMDeclarePackage` call.
`dependency_files`: A list of files that should be additionally loaded.

Each option object has the following layout, all keys must be complete lower-case or upper-case.
```
    {
      "name": "MYLIB_BUILD_TEST",
      "description": "build tests",
      "default": "${PROJECT_IS_TOP_LEVEL}"
    }
```

EAch translationset object has the following layout, all keys must be complete lower-case or upper-case.
```
    {
      "name": "myLib::myLib2",
      "type": "library",
      "language": "cxx_std_20",
      "dependencies": [
         "myLib::myLib"
      ]
    }
```

Each package objects has the following layout, all keys must be complete lower-case or upper-case.
Only the `name` field is mandatory.
```
    {
        "if": "", # some condition with cmake variables
        "name": "name, mandetory",
        "force": false,
        "version": "",
        "git_tag": "",
        "download_only": "",
        "patches": "",
        "github_repository": "",
        "bitbucket_repository": "",
        "gitlab_repository": "",
        "git_repository": "",
        "source_dir": "",
        "find_package_arguments": "",
        "no_cache": false,
        "system": true,
        "git_shallow": false,
        "exclude_from_all": true,
        "source_subdir": "",
        "custom_cache_key": "",
        "url": [],
        "options": [],
        "download_command": [],
        "cpmaddpackage_command": "cpmaddpackage",
        "cmake_commands": []
    }
```

## Helper Variable
Field `git_tag` supports the variable `{VERSION}` which will be replaced by the content of the `version` field.
This allows to write more sophisticated tags like:
```
    {
        "name": "pcre2",
        "version": "10.43",
        "git_tag": "pcre2-{VERSION}"
        ...
    }
```
