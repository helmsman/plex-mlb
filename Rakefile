require 'rake'
require 'rake/clean'
require 'rake/packagetask'
require "tempfile"
require 'yaml'

# base paths
PLEXMLB_ROOT         = File.expand_path( File.dirname( __FILE__ ) )
PLEXMLB_SRC_DIR      = File.join( PLEXMLB_ROOT, 'src' )
PLEXMLB_DIST_DIR     = File.join( PLEXMLB_ROOT, 'dist' )

# paths used by the :install task
PLEX_SUPPORT_DIR     = File.expand_path( '~/Library/Application Support/Plex Media Server' )
PLEX_PLUGIN_DIR      = File.join( PLEX_SUPPORT_DIR, 'Plug-ins' )
PLEX_SITE_CONFIG_DIR = File.join( PLEX_SUPPORT_DIR, 'Site Configurations' )

class File
  def self.binary?(name)
    fstat = stat(name)
    if !fstat.file?
      false
    else
      open(name) do |file|
        blk = file.read(fstat.blksize)
        blk.size == 0 || blk.count("^ -~", "^\r\n") / blk.size > 0.3 || blk.count("\x00") > 0
      end
    end
  end
end

def erb config, file
  temp = Tempfile.new( "erb" )
  File.open( file ).each_line do |line|
    temp << line.gsub(/<%=(.*?)%>/) do
      prop = $1.strip
      if value = config[prop]
        value
      else
        raise "couldn't find property `#{prop}' (in #{file})"
      end
    end
  end
  temp.close

  mv( temp.path, file, :verbose => false )
end

def load_config env=nil
  yaml = YAML.load_file( File.join( PLEXMLB_ROOT, 'config.yml' ) )
  if env
    yaml['default'].merge yaml[env.to_s]
  else
    yaml['default']
  end
end

def rm_if_exists file
  rm_rf file if File.exists? file
end

def site_config_name config
  config['PLUGIN_ID'].split('.').last + '.xml'
end

config = load_config

# files to blow away with a `rake clobber`
CLOBBER.include( PLEXMLB_DIST_DIR )

task :default => :dist

desc 'Build the distribution.'
namespace :dist do
  desc 'Build a dev distribution.'
  task :dev do
    config = load_config :development
    Rake::Task["dist:release"].execute
  end

  desc 'Build a release distribution.'
  task :release do
    rm_if_exists File.join( PLEXMLB_DIST_DIR )
    bundle_dest = File.join( PLEXMLB_DIST_DIR, "#{config['PLUGIN_NAME']}.bundle" )
    mkdir_p( bundle_dest )
    cp_r File.join( PLEXMLB_SRC_DIR, 'bundle' ), File.join( bundle_dest, 'Contents' )
    cp_r File.join( PLEXMLB_SRC_DIR, 'site configuration.xml' ), File.join( PLEXMLB_DIST_DIR, site_config_name( config ) )

    # process files with erb
    FileList[ File.join( PLEXMLB_DIST_DIR, '**', '*' ) ].each do |file|
      erb config, file unless ( File.directory?( file ) || File.binary?( file ) )
    end
  end
end
task :dist => 'dist:release'

desc 'Install the bundle'
task :install => 'dist:dev' do
  rm_if_exists File.join( PLEX_PLUGIN_DIR, "#{config['PLUGIN_NAME']}.bundle" )
  rm_if_exists File.join( PLEX_SITE_CONFIG_DIR, site_config_name( config ) )

  cp_r File.join( PLEXMLB_DIST_DIR, "#{config['PLUGIN_NAME']}.bundle" ), File.join( PLEX_PLUGIN_DIR, "#{config['PLUGIN_NAME']}.bundle" )
  cp_r File.join( PLEXMLB_DIST_DIR, site_config_name( config ) ), File.join( PLEX_SITE_CONFIG_DIR, site_config_name( config ) )
end
