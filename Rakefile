require 'erb'
require 'json'
require 'pathname'

require 'bundler/setup'

require 'tilt'

module FormatHelper
  def hms(sec)
    min, sec = sec.divmod(60)
    hour, min = min.divmod(60)
    if hour.zero?
      '%02d:%02d' % [min, sec]
    else
      '%d:%02d:%02d' % [hour, min, sec]
    end
  end
end

class Book
  attr_reader :chapters

  def initialize(chapters:)
    @chapters = chapters
  end

  def duration
    chapters.map(&:duration).sum
  end

  def to_s
    chapters.map(&:to_s).join("\n\n")
  end

  def to_html
    chapters.map(&:to_html).join("\n\n")
  end
end

class Chapter
  attr_reader :number, :sections

  def initialize(number:, sections:)
    @number = number
    @sections = sections
  end

  def duration
    sections.map(&:duration).sum
  end

  def to_s
    sections.map do |section|
      section.to_s(self)
    end.join("\n\n")
  end

  def to_html
    sections.map do |section|
      section.to_html(self)
    end.join("\n\n")
  end
end

class Section
  include FormatHelper

  attr_reader :number, :sub_sections

  def initialize(number:, sub_sections:)
    @number = number
    @sub_sections = sub_sections
  end

  def duration
    sub_sections.map(&:duration).sum
  end

  def to_s(parent)
    lines = ["#{parent.number}-#{number} (#{hms(duration)} / #{hms(parent.duration)})"]

    lines += sub_sections.map(&:to_s)

    lines.join("\n\n")
  end

  def to_html(parent)
    lines = ["<h3>#{parent.number}-#{number} (#{hms(duration)} / #{hms(parent.duration)})</h3>"]

    lines += sub_sections.map(&:to_html)

    lines.join("\n\n")
  end
end

class SubSection
  attr_reader :number, :tracks

  def initialize(number:, tracks:)
    @number = number
    @tracks = tracks
  end

  def duration
    tracks.map(&:duration).sum
  end

  def to_s
    tracks.map do |track|
      track.to_s(self)
    end.join("\n")
  end

  def to_html
    lines = ['<ul>']

    lines += tracks.map do |track|
      track.to_html(self)
    end

    lines += ['</ul>']

    lines.join("\n")
  end
end

class Track
  include FormatHelper

  attr_reader :number, :basename, :duration

  def initialize(number:, basename:, duration:)
    @number = number
    @basename = basename
    @duration = duration
  end

  def to_s(parent)
    if parent.tracks.one?
      "  * #{basename}  (#{hms(duration)})"
    else
      "  * #{basename}  (#{hms(duration)} / #{hms(parent.duration)})"
    end
  end

  def to_html(parent)
    if parent.tracks.one?
      "<li>#{basename} (#{hms(duration)})</li>"
    else
      "<li>#{basename} (#{hms(duration)} / #{hms(parent.duration)})</li>"
    end
  end
end

class DurationParser
  def initialize(cache_path)
    @cache_path = cache_path
    @cache = @cache_path.exist? ? JSON.parse(@cache_path.read) : {}
  end

  def parse(path)
    path = path.to_s

    value = @cache[path]

    return value if value

    warn path

    value = _parse(path)

    @cache[path] = value
    @cache_path.write(JSON.pretty_generate(@cache))

    value
  end

  private

  def _parse(path)
    info = IO.popen(['ffmpeg', '-i', path, :err => [:child, :out]], &:read)

    if /Duration: (\d+):(\d+):(\d+\.\d+)/ =~ info
      hour = $1.to_i
      min = $2.to_i
      sec = $3.to_f.ceil
      (hour * 60 * 60) + (min * 60) + sec
    else
      warn info
      raise 'invalid info'
    end
  end
end

DURATION_PARSER = DurationParser.new(Pathname.new('durations.json'))

def build_book(book_dir)
  track_paths = Pathname.glob("#{book_dir}/**/*.mp3").sort

  items = track_paths.map do |track_path|
    basename = track_path.basename(track_path.extname).to_s

    if /\A(\d)(\d|X)(\d)(\d)(\d)N?\z/ =~ basename
      {
        book_number: Integer($1),
        chapter_number: $2 == 'X' ? 10 : Integer($2),
        section_number: Integer($3),
        sub_section_number: Integer($4),
        track_number: Integer($5),
        path: track_path,
        basename: basename,
        duration: DURATION_PARSER.parse(track_path)
      }
    else
      raise "invalid basename: #{basename}"
    end
  end.reject do |item|
    item.fetch(:chapter_number) == 0
  end

  chapters = items.group_by do |item|
    item.fetch(:chapter_number)
  end.map do |chapter_number, chapter_items|
    sections = chapter_items.group_by do |chapter_item|
      chapter_item.fetch(:section_number)
    end.map do |section_number, section_items|
      sub_sections = section_items.group_by do |section_item|
        section_item.fetch(:sub_section_number)
      end.map do |sub_section_number, sub_section_items|
        tracks = sub_section_items.map do |item|
          Track.new(
            number: item.fetch(:track_number),
            basename: item.fetch(:basename),
            duration: item.fetch(:duration)
          )
        end

        SubSection.new(number: sub_section_number, tracks: tracks)
      end

      Section.new(number: section_number, sub_sections: sub_sections)
    end

    Chapter.new(number: chapter_number, sections: sections)
  end

  Book.new(chapters: chapters)
end

class RenderingContext
  include ERB::Util
end

def erb(template_path, locals = {}, &block)
  template = Tilt.new(template_path, trim: '-')
  template.render(RenderingContext.new, locals, &block)
end

task :default => :build

task :build do
  book_dirs = %w[N_Math1A N_Math2B N_Math3]

  brand = 'Nagaoka Math Book'

  book_dirs.each do |book_dir|
    book = build_book(book_dir)

    File.write("#{book_dir}.txt", book.to_s)

    html = erb('views/layout.erb', title: book_dir, brand: brand) do
      erb('views/show.erb', body: book.to_html)
    end

    File.write("#{book_dir}.html", html)
  end

  html = erb('views/layout.erb', title: brand, brand: brand) do
    paths = book_dirs.map {|v| "#{v}.html" }
    erb('views/index.erb', paths: paths)
  end

  File.write('index.html', html)
end
