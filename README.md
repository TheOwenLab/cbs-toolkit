# CBS Python
# Description
This Docker image contains a Python environment with custom packages installed for working with Cambridge Brain Sciences (CBS) data. Consider this Docker image an alternative to having to install and maintain a Python development environment; instead, just spin up a container and make use of all the Python-ic goodness stored inside! You can preprocess your CBS datasets to extract score features, re-organize the data, calculate norms, calculate domain scores, encrypt your data files, and other things. These functions are all accessed using "Script Entrypoints" - see below for descriptions. Note, these tools do not include any statistics or analysis routines! You'll have to use your software of choice (R, SPSS, Python, etc.) to read and analyze the data saved by the commands you run here.

Why use Docker for this stuff? Well, it makes it easy to distribute these tools for others to use without having to worry about what kind of computer they're using, or what specific Python version, or package versions, are installed. Better yet, you don't even have to know how to use Python in order to use all these Python scripts! Also, it's easy to distribute updates for these tools without having to worry about dependencies, etc; Docker just pulls the latest image when a new one is available. There are lots of reasons! Go forth and Google...

# Script Entrypoints
The toolkit provided in this Docker image is really just a collection of Python scripts (kinda) that process your input files (e.g., a raw CBS data export in .csv form) and save some output files. For any of the following entrypoints<sup>1</sup>, you can see the complete set of arguments, their descriptions, defaults, etc. by using the `--help` argument. For example: `cbs_parse_data --help`.

## cbs_parse_data
The script takes as an input a raw CBS .csv data export and parses/extracts all kinds of score feature data and saves them into a nice wideform format with one row per assessment<sup>2</sup>. 

## cbs_score_calculator
Parses the data like the previous script, but also:
 - generates age/gender matched norms for each participant in your dataset,
 - z-scores all test scores and their features,
 - calculates "domain" scores based on the Varimax-rotated PCA loadings from Hampshire et al. 2012,
 - generates age/gender matched norms for the domain scores for each of your participants,
 - z-scores the domain scores.

## cbs_encrypt
Batch encryption of files using Fernet encryption - which is an implementation of symmetric (aka "secret key") authenticated cryptography. Basically, super securely encrypt data files. More information to added...

## cbs_decrypt
Batch decrypt files that have been encrypted by `cbs_encrypt`. More information to added...

### Footnotes
1. They're called "script entrypoints" (rathen than just scripts) because you are overriding the default command that is executed when the container is launched; hence, they're like "entrypoints" to the running container.
2. An "assessment" is a set of CBS scores belonging to a participant at a given time. Typically, this would be one "time point" of study. Scores can be linked by a variable called `session_id` or `time_point` or `report`. An assessment can contain any number of tasks, and that depends on the study design and trial setup. Different "batches" can have different tasks.

# Installation
Given that we're doing all this with Docker, installation is trivial!
1. Make sure that you have [Docker](https://www.docker.com/) installed.
1. Test your installation: `docker run hello-world`.
1. When you run any CBS Python tasks, Docker will check if there have been updates and re-pull the image if needed.

# Instructions
This toolkit was initially designed to pre-process CBS data in a [Datalad](https://www.datalad.org/) pipeline to ensure computational reproducibility. See [datalad-containers](https://docs.datalad.org/projects/container/en/latest/) and the [Datalad handbook](https://handbook.datalad.org/en/latest/basics/101-133-containersrun.html). Alternatively, you just can just use this image to manually spin up a Docker container and process your data. See the appropriate Instructions depending on your use case. Either way, you have to use the terminal (e.g., on [MacOS](https://iterm2.com/)) to use these tools. That's right - no GUI for *you*!

## Datalad
- *TBI*

## Manually
1. Navigate to the folder containing your raw CBS data export (in a .csv form) (e.g., `cd ~/Documents/myproject/data/`)
1. Run the required script in a container: `docker run --rm -it -v $PWD:/tmp -w /tmp conorwild/cbspython:latest <entrypoint arg1 arg2 ...>`; replace `<entrypoint arg1 arg2 ...>` with the desired entrypoint name and the arguments (don't include the `<`/`>`!)

For example, try this: `docker run --rm -it -v $PWD:/tmp -w /tmp conorwild/cbspython:latest cbspython --help`

Ok, the command above:
 - creates and runs a new container ("`docker run`"), 
 - based on the latest cbspython image ("`conorwild/cbspython:latest`"; will auto-fetch if an update is available),
 - mounts your current working directory to `/tmp` ("`-v $PWD:/tmp`") ,
 - sets the containers working directory to that folder ("`-w tmp`")
 - and executes the remaining stuff (i.e., the python script with supplied arguments) within the container's environment.


# Notes for Myself

## Build
To build this image, you need to have SSH access to CBS [Bitbucket](https://bitbucket.org/cambridgebrainsciences/cbspython/src/main/) and the [Owenlab GIN Server](http://owenlab.ssc.uwo.ca/). That is, you must have an account on each hosting site with your public SSH key added to the profile in each account.
```
# From the cbs-sci-containers root directory
ssh-add ~/.ssh/id_rsa
datalad run -m "Rebuilding cbspython Docker image" -o cbspython/image/ "cd cbspython && ./build.sh"
```

# Debug
Run a bash shell: `docker run --rm -it --entrypoint /bin/bash cbspython:latest`

# Test
```
$ datalad containers-run -m "Testing CBSPython parsing" -n cbspython -i test/test_data.csv "cbs_parse_data {inputs} test_data_out -o stdout --user-tfm 'lambda x: x[:x.index(\"@\")]'"
```

# Reset
```
git reset --hard # removes staged and working directory changes
git clean -f -d # remove untracked
```
