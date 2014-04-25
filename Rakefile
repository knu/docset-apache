require 'bundler/setup'
Bundler.require

require 'find'
require 'uri'

def Version(string)
  Gem::Version.new(string)
end

def doclink(url, anchor = nil)
  URI(url.to_s).tap { |uri|
    uri.fragment = URI.encode_www_form_component(anchor) if anchor
  }
end

def web
  @web ||= Mechanize.new
end

DOCSET = 'Apache.docset'
DOCSET_ARCHIVE = File.basename(DOCSET, '.docset') + '.tgz'
DEFAULT_VERSION = Version(File.read('VERSION').chomp)
DEFAULT_LANGUAGE = 'en'
DOCS_URI = URI('http://www.apache.org/dyn/mirrors/mirrors.cgi/httpd/docs/')
FILENAME_FORMAT = 'httpd-docs-%s.en.zip'
RE_FILENAME = FILENAME_FORMAT.match(/%s/).tap { |m|
  break /\A#{Regexp.quote(m.pre_match)}(?<version>\d+(?:\.\d+)*)#{Regexp.quote(m.post_match)}\z/
}

desc 'Check for update and bump VERSION if a newer version is found.'
task :check do |t|
  dir = web.get(DOCS_URI).link_with(css: '#content a[href]').click()
  latest = dir.links_with(text: RE_FILENAME).each_with_object({}) { |link, memo|
    version = Version(RE_FILENAME.match(link.text)[:version])
    if !(max = memo[:max]) || max < version
      memo[:max] = version
    end
  }[:max]

  if latest && latest > DEFAULT_VERSION
    puts 'Found httpd-docs %s' % latest
    File.write('VERSION', "%s\n" % latest)
  end
end

desc 'Fetch the Apache document zip file.'
task :fetch, [:version] do |t, args|
  version = args[:version] ? Version(args[:version]) : DEFAULT_VERSION

  filename = FILENAME_FORMAT % version

  if File.file?(filename)
    puts '%s already exists.' % filename
  else
    uri = DOCS_URI + filename
    zip_uri = web.get(uri).link_with(css: '#content a[href]').resolved_uri

    puts 'Downloading %s' % zip_uri
    web.download(zip_uri, filename)
  end
end

desc 'Build a docset in the current directory.'
task :build, [:version] do |t, args|
  version = args[:version] ? Version(args[:version]) : DEFAULT_VERSION

  filename = FILENAME_FORMAT % version
  target = DOCSET
  docdir = File.join(target, 'Contents/Resources/Documents')

  Rake::Task['fetch'].invoke(version.to_s) unless File.file?(filename)

  rm_rf target

  mkdir_p docdir

  cp 'Info.plist', File.join(target, 'Contents')

  # Viva bsdtar!
  sh 'tar', '-xf', filename, '--strip-components=1', '-C', docdir

  # Icon (64x64)
  icon = File.join(target, 'icon.png')
  size = 64
  ssize = size.to_s

  sh 'sips', '--setProperty', 'format', 'png',
     File.join(docdir, 'images/feather.gif'),
     '--out', icon
  sh 'sips', '--rotate', '-30', icon
  sh 'sips', '--resampleHeightWidthMax', ssize, icon
  sh 'sips', '--padToHeightWidth', ssize, ssize, icon

  # Index
  db = SQLite3::Database.new(File.join(target, 'Contents/Resources/docSet.dsidx'))

  db.execute(<<-SQL)
    CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);
    CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);
  SQL

  insert = db.prepare(<<-SQL)
    INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?, ?, ?);
  SQL

  puts 'Indexing documents'

  Dir.chdir(docdir) {
    Dir.glob('**/*.html') { |path|
      doc = Nokogiri::HTML(File.read(path))

      if h1 = doc.at('//h1[starts-with(text(), "Apache Module ")]')
        name = h1.text[/\AApache Module (.+)/, 1]
        uri = doclink(path, (anchor = h1.at('./ancestor-or-self::*[@id]/@id')) && anchor.to_s)
        insert.execute(name, 'Module', uri.to_s)
      end

      doc.xpath('//div[@class="directive-section"]/h2/a[@id][1]').each { |a|
        name = a[:id]
        uri = doclink(path, a[:id])
        insert.execute(name, 'Directive', uri.to_s)
      }
    }
  }
end

desc 'Archive the generated docset into a tarball'
task :archive do
  sh 'tar', '-zcf', DOCSET_ARCHIVE, '--exclude=.DS_Store', DOCSET
end

task :default => :build
