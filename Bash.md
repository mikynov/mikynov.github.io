# Consistent Bash scripting

## Naming

Following [Google's Shell Style Guide](https://google.github.io/styleguide/shellguide.html#s2.1-file-extensions):

* Executable scripts should **not have an extension**
* Sourced scripts (libraries) **must have an `.sh`** extension and no executable flag

## Boilerplate

    #!/usr/bin/env bash

    readonly TRACE="${TRACE:-}"
    [[ -n "${TRACE}" ]] && set -o xtrace
    set -o errexit
    set -o errtrace
    set -o nounset
    set -o pipefail
    set -o noclobber
    IFS=$'\n\t'

    SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
    # shellcheck disable=SC2034
    readonly SCRIPT_DIR

    _main() {
        :
    }

    [[ "${BASH_VERSINFO[0]}" -lt 4 ]] && echo "Requires Bash >= 4" && exit 44
    [[ "${BASH_SOURCE[0]}" == "${0}" ]] && _main "${@}"

## Idioms

### Shebang

    #!/usr/bin/env bash

### Script directory
    
    readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

### Enforce Bash version

    [[ "${BASH_VERSINFO[0]}" -lt 4 ]] && echo "Requires Bash >= 4" && exit 44

### Filename and extension

    readonly FILE=$(basename "${0}")
    filename=${FILE%.*}
    extension=${FILE##*.}

### Letter case

    readonly STR="BfLmPsVz"
    uppercase=${STR^^}
    lowercase=${STR,,}

### Remove newline

    readonly STR=$'text\r'
    echo "${STR//[$'\t\r\n']}"

### Cleanup

    function _cleanup {
        :
    }
    trap _cleanup EXIT

### Arrays

**Pass**

    function give_me_array() {
        local arr=("${@}")
        shift
    }

**Join**

    arr=("a" "b" "c d e")
    (IFS=","; echo "${arr[*]}")

    arr=("a" "b" "c d e")
    printf -v joined_str '%s,' "${arr[@]}"
    echo "${joined_str%,}"

**Split**

    arr=("a" "b" "cd e")
    printf -v joined_str '%s,' "${arr[@]}"
    echo "${joined_str%,}"

    arr=("a" "b" "cd e")
    (IFS=","; echo "${arr[*]}")

    joined_str="a,b,cd e"
    IFS="," read -r -a arr <<< "${joined_str}"
    echo "${arr[*]}"

### Parse args

Parse arguments and set values to `readonly` variables (`A_OPT`, `B_PARAM`).  
Positional parameters are set in `_POSITIONAL_PARAMS` array.

    _parse_args() {
        declare -a -g _POSITIONAL_PARAMS

        while [[ $# -gt 0 ]]; do
            case "$1" in
                -h|-\?|--help)
                    _help
                    exit 1
                    ;;
                -a|--a-opt)
                    readonly A_OPT="1"
                    shift
                    ;;
                -b|--b-param)
                    if [[ -z "${2-}" || "${2:0:1}" == "-" ]]; then
                        echo "ERROR: Undefined value for ${1}" >&2
                        exit 1
                    else
                        readonly B_PARAM="${2}"
                        shift 2
                    fi
                    ;;
                --b-param=?*)
                    readonly B_PARAM="${1#*=}"
                    shift
                    ;;
                --)
                    shift
                    break
                    ;;        
                -*)
                    echo "ERROR: Unknown argument ${1}" >&2
                    exit 1
                    ;;
                *)
                    _POSITIONAL_PARAMS+=("${1}")
                    shift
                    ;;
            esac
        done
        # shellcheck disable=SC2294
        eval set -- "${_POSITIONAL_PARAMS[@]}"
    }

**Usage**

    _main() {
        _parse_args "${@}"
        # Set positional parameters to function(!) scope
        # shellcheck disable=SC2294
        eval set -- "${_POSITIONAL_PARAMS[@]}"
        # Access named params
        [[ -n ${A_OPT:-} ]] && echo "Flag is set"
        [[ -n ${B_PARAM:-} ]] && echo "B value is ${B_PARAM}"
        # Access positional params
        for ((i=1; i<=${#}; i++)); do
            [[ -n ${i:-} ]] && echo "\${${i}}=${!i}"
        done
    }

**Credits**

* [How do I parse command line in Bash?](https://stackoverflow.com/a/14203146) on *SO*
* [Bash: Argument Parsing](https://medium.com/@Drew_Stokes/bash-argument-parsing-54f3b81a6a8f)
* [How can I handle command-line easily?](http://mywiki.wooledge.org/BashFAQ/035) from *BashFAQ*

### Help

    _help() {
        cat <<EOF
    NAME
        ${0} - Lorem ipsum dolor sit amet

    SYNOPSIS
        ${0} [ARGS ...] [TARGET]

    DESCRIPTION
        Ut enim ad minim veniam, quis nostrud exercitation ullamco 
        laboris nisi ut aliquip ex ea commodo consequat.

        -a, --a-option         Duis aute irure
        -b, --b-param=VALUE    Excepteur sint occaecat

    EXAMPLES
        Mattis rhoncus urna neque viverra justo nec ultrices:

            ${0} -a -b 10 /tmp/target

        A pellentesque sit amet porttitor eget dolor morbi non:

            ${0} --b-param=10

    EOF
    }

### Check if running inside a container

    _in_container() {
        [[ -f /proc/self/cgroup ]] && grep -q "docker" /proc/self/cgroup
    }

**Usage**

    _main() {
        if _in_container; then
            :
        else
            :
        fi

        _in_container && echo "Cargo"
    }

## Tools

* [Shellcheck](https://github.com/koalaman/shellcheck) and its Visual Studio Code [extension](https://marketplace.visualstudio.com/items?itemName=timonwong.shellcheck)
* [Shellharden](https://github.com/anordal/shellharden) can translate a Bash source to conforms the best practices (*Shellcheck*)

## Docker way

### Bash test environment

    docker run -it --rm --volume "$(pwd):/home/bash" bash:4

Check tags for other Bash versions at [Docker hub](https://hub.docker.com/_/bash).

### Shellcheck

    docker run --rm --volume "$(pwd):/mnt" koalaman/shellcheck:stable /mnt/SCRIPT.sh

### Shellharden

    docker run --rm --volume "$(pwd):/mnt" sbkg0002/shellharden:4.0 /mnt/SCRIPT.sh

## Sources

* [Google's Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
* [Bash Notes for Professionals](https://goalkicker.com/BashBook/BashNotesForProfessionals.pdf)
* [Safe ways to do things in Bash](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md) from *Shellharden*
* [String manipulation](http://wiki.juneday.se/mediawiki/index.php/Bash_idioms#String_manipulation) from *Bash idioms*
* [Gallery of bad code](https://github.com/koalaman/shellcheck#gallery-of-bad-code) from *Shellcheck*
* [Bash Pitfalls](http://mywiki.wooledge.org/BashPitfalls)
* [Bash Strict Mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/)
