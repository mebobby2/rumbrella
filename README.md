# Rumbrella

The Umbrella App for the Rumbl application

## Install

1. Go into apps/info_sys and run ```mix deps.get```
2. Go into apps/rumbl/ and run ```mix deps.get```
3. Go into apps/rumbl/ and run ``npm install```

## Run

1. From root of this project, run ```mix phoenix.server```

## Notes

### Git pull
Whenever you make new changes to a submodule, you have to pull the latest changes from the submodule here. Use the command
```
git submodule update --recursive --remote
```
And then push the new changes as a commit.