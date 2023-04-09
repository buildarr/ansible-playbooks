# Buildarr example Ansible playbooks

A collection of Ansible playbooks and roles that use Docker Compose and Buildarr to automatically deploy, configure and manage Arr applications.

Most people will probably already have their own way of deploying Arr stack containers or services. These playbooks are meant to be a demonstration of how you can use Buildarr in combination with the traditional configuration management tools to not just automate deployment of your Arr application infrastructure, but also automate configuration of them, so it is ready to run with just a single command.

These playbooks show how you can implement:

* Full deployment and configuration management, starting a brand new, fully setup cluster from scratch with just one command
* Creating and mounting service configuration and data folders with the correct permissions, and support for hardlinks
* Templating the Buildarr configuration file, **including automatic retrieval of API keys**
* Deploying the Arr stack applications, and starting Buildarr to configure them *after* ensuring they are accessible

## Prerequisites

On the Ansible host machine:

* Ansible (any version as long as it is relatively recent)
* The `community.docker` Ansible collection (can be installed using `ansible-galaxy install -r requirements.yml`)
* Passwordless SSH access to the target machine (e.g. using an SSH key loaded into SSH agent)

On the target machine:

* Python 3 (the required version depends on the version of Ansible used on the host)
* Docker (version 18.06 or later)
* Docker Compose (version 1.22.0 or later)

## Arr Stack

This is an Ansible playbook for deploying a full stack of Arr applications, managed by Buildarr.

### Structure

The following services are deployed in this playbook:

* Three Sonarr instances configured for specific types of content:
    * SD/HD TV shows (`sonarr-hd`)
    * 4K TV shows (`sonarr-4k`)
    * Anime series (`sonarr-anime`)
* One Prowlarr instance (`prowlarr`) for indexer management, fully integrated with each Sonarr, with the following indexers configured:
    * `1337` (for TV shows)
    * `nyaa.si` (for anime)
* One FlareSolverr instance (`flaresolverr`) for proxying Prowlarr indexer requests to `1337`
* One Transmission instance (`transmission`) for downloading releases, configured on each Sonarr and Prowlarr
* One Buildarr instance (`builarr`) for managing the configuration for each Sonarr and Prowlarr

By default the Arr stack is deployed to `/opt/arrstack`, with the below folder structure. This is partially based on the [TRaSH-Guides guide](https://trash-guides.info/Hardlinks/How-to-setup-for/Docker) for configuring an Arr stack environment for hardlinking.

* `/opt/arrstack` - Environment directory (configurable as `arrstack_env_dir`)
    * `data` - Data directory for media (configurable as `arrstack_data_dir`)
        * `media` - Media destination directory
            * `shows` - TV shows (both SD/HD and 4K, with SD/HD ones having `- Default` appended to the filename)
            * `anime` - Anime TV shows
        * `torrents` - Torrent download directory
            * `shows` - TV show torrents
                * `hd` - HD TV show torrents
                * `4k` - 4K TV show torrents
                * `anime` - Anime series torrents
    * `buildarr` - Buildarr configuration directory/volume
        * `buildarr.yml` - Buildarr base configuration file
        * `sonarr.yml` - Buildarr Sonarr instance configuration (included by `buildarr.yml`)
        * `prowlarr.yml` - Buildarr Prowlarr instance configuration (included by `buildarr.yml`)
    * `sonarr-{hd,4k,anime}` - Sonarr configuration volume
    * `prowlarr` - Prowlarr configuration volume
    * `transission` - Transmission configuration volume
    * `docker-compose.yml` - Arr stack Docker Compose file

The application containers all have ports mapped to `localhost` to allow access from outside the container network on the host machine only:

* `sonarr-hd`: `http://localhost:8989`
* `sonarr-4k`: `http://localhost:8990`
* `sonarr-anime`: `http://localhost:8991`
* `prowlarr`: `http://localhost:9696`
* `flaresolverr`: `http://localhost:8191`
* `transmission`: `http://localhost:9091`

### Configuration

The defaults for the Ansible role are stored in `roles/arrstack/defaults/main.yml`.

Any configuration parameter not located here are generally located in the template files, and can be modified before deployed:

* `roles/arrstack/templates/docker-compose.yml.j2` - Arr stack Docker Compose file template
* `roles/arrstack/templates/buildarr.yml.j2` - Buildarr base configuration file template
* `roles/arrstack/templates/sonarr.yml.j2` - Buildarr Sonarr instance configuration template
* `roles/arrstack/templates/prowlarr.yml.j2` - Buildarr Prowlarr instance configuration template

### Installation

To deploy the Arr Stack on a host of your choice, simply run the following command, inputting the hostname or IP address of the machine to deploy to.

```bash
# Note that the comma after the host address is required!
ansible-playbook -i "<target host address>," arrstack.yml
```

Ansible will take care of everything from creating the files and folders, to starting the containers and kicking off Buildarr to conigure them.

### Usage

Once deployment is complete, you can open the Sonarr instances in your browser and use them as you normally would:

* Sonarr for SD/HD TV shows: http://localhost:8989
* Sonarr for 4K TV shows: http://localhost:8990
* Sonarr for anime series: http://localhost:8991

When adding sereis to any Sonarr instance, they will communicate with Prowlarr to search for releases, and then set them for download on Transmission.

Once done, the Sonarr instances will hardlink the downloaded files to the destination media folders, making them available for any Jellyfin/Emby/Plex instance watching them.

### Troubleshooting

#### I am getting indexer errors on the Sonarr instances

To reduce the risk of getting blocked from the configured public trackers, Prowlarr is configured to limit each indexer to 256 searches, and 26 grabs, every rolling 24 hours.

This is intended to give you some room to test if the deployment is working, but may not be enough to actually use it.

These values can either be set in the templates themselves, or changed using the `arrstack_prowlarr_query_limit` and `arrstack_prowlarr_grab_limit` Ansible facts.

#### Sonarr is not downloading files/not downloading the right files

The configuration for the Sonarrs is mostly based off of the [TRaSH-Guides](https://trash-guides.info/Sonarr) recommended defaults, and there is a high chance that it is not optimal for your use case.

Prowlarr is also configured to require at least 5 seeders for a grab, which either may not be enough, or too much for some less popular series.

Try changing the configuration for the Sonarrs/Prowlarr in the Buildarr configuration file to better match your needs.

Once the changes are redeployed to the host using the Ansible command, Buildarr will automatically pick up the changess and update the Sonarr instances.

If you have any suggestions for better configuration defaults for the Ansible playbook, please let me know by creating a GitHub issue or pull request!
