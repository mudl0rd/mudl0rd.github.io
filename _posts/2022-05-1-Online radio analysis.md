---
layout: post
published: true
title: Online radio stream analysis for fun and profit
---
# tldr;

I have made 2 tools for analysing data from online radio stations which can then be used for
further purposes. This was done entirely as an educational exercise.

## Overview

For some time, I argued that some radio stations, with my work colleagues, that some certain
radio stations we listen to, have a tendancy to repeat certain songs. 

One of which is a Western Sydney radio station, aptly named [WSFM](https://www.wsfm.com.au/).

We would often debate as to the "quality" of such radio stations. So one day I decided, scientifically,
to judge how often certain songs get repeated as well as the type of content broadcast.

## Process

This is done through a two tier process:

* Actually logging for a time period the actual songs that are broadcast.

This had the added side effect of

* Polling web APIs such as ones offered by [Deezer](www.deezer.com) for more song information.

Logging the songs involved developing a [foobar2000](https://www.foobar2000.org) component to log streams as they
are played, this was done over a period of 7 days.

```
#include "stdafx.h"
#include <iostream>
#include <fstream>
#include <filesystem>
#include <map>
using namespace pfc;
using namespace std;
/*
Compile using the FB2K SDK.
Based around the foobar2000 sample component.
Should be easy enough to go from there.
Uses C++17 because why the fuck not!
--mudlord
*/
namespace {
	

	class play_callback_psc : public play_callback_static {
		std::map<std::string, int> songmap;
	private:
		void song_changed(pfc::string songtit)
		{
			std::filesystem::path path = std::filesystem::current_path() / "stream.txt";
			ofstream file_out;
			std::map<std::string, int>::iterator it = songmap.find(songtit.c_str());
			if (it == songmap.end())
				songmap.emplace(songtit.c_str(), 1);
			else
				++songmap[songtit.c_str()];
			file_out.open(path, std::ios_base::binary);
			for (auto& x : songmap)
			file_out << '"' << x.first << '"' << "," << x.second << endl;
		}
	public:
		void on_playback_dynamic_info_track(const file_info& p_info)
		{
			const char* artist = p_info.meta_get("artist", 0);
			if (artist == NULL)return;
			const char* title = p_info.meta_get("title", 0);
			if (title == NULL)return;
			pfc::string art = artist;
			pfc::string title1 = title;
			pfc::string songtit = art + " - " + title1;
			if (!art.is_empty())
				song_changed(songtit);
		}
		unsigned get_flags() override {
			return flag_on_playback_dynamic_info_track | flag_on_playback_starting |
				flag_on_playback_stop;
		}
		void on_playback_stop(play_control::t_stop_reason p_reason) override {
			// Terminate the current stream
		}
		void on_playback_starting(play_control::t_track_command p_command, bool p_paused) override {
			
		}
		void on_playback_new_track(metadb_handle_ptr p_track) override {

		}
		void on_playback_seek(double p_time) override {}
		void on_playback_pause(bool p_state) override {}
		void on_playback_edited(metadb_handle_ptr p_track) override {}
		void on_playback_dynamic_info(const file_info& p_info) override {
		}
		
		void on_playback_time(double p_time) override {}
		void on_volume_change(float p_new_val) override {}
	};
	FB2K_SERVICE_FACTORY(play_callback_psc);
```

Sifting through the data of the component was done with a external tool, the source code at the end of this article. From there, CSV files can be used to peruse the data in a spreadsheet, whereas information on the song such as artist, title, album, record label, and other things could easily be logged using an outside tool to correlate info to find tracks linking all this.

## Problems

There was several problems:

* Getting the foobar2000 component to compile and grabbing song information.

This was a problem due to various linking errors due to the compiler used, downgrading to a MSVC version of 2019 solved. Grabbing the song info was a problem due to erroneous usage of the playlist_callback API, but this was eventually fixed.

* Working with Meson to link the outside data processing tool with libcurl.

One of the problems in development was messing around with Meson to properly link and compile the tool, which would be used for further data analysis. In the end, a Meson buildfile was used as follows

```
project('generic', 'cpp',default_options: ['buildtype=debug','c_std=c11', 'cpp_std=c++17'])
inc = include_directories('src')
curllib = dependency('libcurl',
  required: true,
  static: false,
  method: 'pkg-config')
glob = run_command('python3', 'src_batch.py')
src = glob.stdout().strip().split('\n')
dir_base = meson.current_source_dir()
dir_install = join_paths(dir_base, 'compile_dir')
executable('tool',install: true,install_dir: dir_install,dependencies : curllib,
 include_directories : [inc],sources : [src])
 ```

 * Working with CSV parsing

 Many different libraries used for CSV parsing were used. Eventually after wrangling with many different libraries, it was decided that a homebrewn format is used to cook the data for use, and a process of trial-and-error were used to format the data into a form thats usable.

 * Deezer API polling

 Some initial frustration was found with usage of the Deezer web API. Messing with libcurl, URL encoding, as well as other things like Deezer API rate limits were an issue but eventually mitigated.

 * Internet reliability.

 Due to weather, in some cases the data dumps were incomplete for a seven day period.


## Source code and data dumps.

[Source repo.](https://github.com/mudlord/radiostream_tools)

A modern FB2K is needed to compile the foobar2000 logging component.
Meson, MSYS2, and Visual Studio Code are needed to compile the Deezer logging tool.

The *.stdb files are CSV files which can be read in spreadsheet applications like LibreOffice's.