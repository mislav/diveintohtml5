# encoding: utf-8
require 'rake/clean'
require 'fileutils'
CLOBBER.include 'build'

WEB_PREFIX = ENV.fetch 'WEB_PREFIX', 'http://mislav.uniqpath.com/diveintohtml5/'
PRINCE = ENV.fetch 'PRINCE', 'prince'

CHAPTERS = %w[
  index.html
  introduction.html
  past.html
  detect.html
  semantics.html
  canvas.html
  video.html
  geolocation.html
  storage.html
  offline.html
  forms.html
  extensibility.html
  history.html
  everything.html
  about.html
  ]

add_class = ->(elem, name) {
  classes = elem['class'].to_s.split(' ')
  classes << name unless classes.include? name
  elem['class'] = classes.join(' ')
  elem
}

dehtml5 = ->(doc) {
  doc.search('article, section, figure, figcaption, hgroup, mark').reverse.each do |elem|
    type = elem.name
    type = 'caption' if type == 'figcaption'
    elem.name = type == 'mark' ? 'span' : 'div'
    add_class.(elem, type)
  end
  doc
}

HEAD_ELEMENTS = %w[ meta title link style script ]

body_element = ->(node) {
  node.elem? and not HEAD_ELEMENTS.include?(node.name)
}

check_code = check_block = nil

check_block = ->(block) {
  block.name == 'pre' && (code = block.at_xpath('./code')) ? check_code.(code) :
    block.element_children.empty?
}
check_code = ->(code) {
  check_block.(code) and code.parent.children.all? { |el| el == code or el.blank? }
}

syntax_highlighter = ->(doc) {
  blocks = doc.search('pre, #everything dd > code, p > code')
  elements = []
  to_colorize = []

  blocks.each do |pre|
    if pre.name == 'code' ? check_code.(pre) : check_block.(pre)
      code = pre.text
      lang = case code
      when /\A\s*</ then 'html'
      when /^\s*(function|return|if|interface|var)\b/,
           /\b(document|window|navigator)[.\[]/,
           /\b(typeof|setAttribute|history\.pushState)\b/
        'js'
      end
      if lang
        elements << (pre.name == 'code' && pre.parent.name == 'p' ? pre.parent : pre)
        to_colorize << [lang, code.rstrip]
      end
    end
  end

  if to_colorize.any?
    # watch out for https://github.com/github/albino/pull/10
    require 'albino/multi'
    Albino::Multi.colorize(to_colorize).each_with_index do |pretty, idx|
      old_code = elements[idx]
      inline_styles = old_code['style']
      # manually initializing DocumentFragment due to bug with encodings
      nodes = old_code.replace Nokogiri::HTML::DocumentFragment.new(doc.document, pretty)
      nodes.at_xpath('.//pre')['style'] = inline_styles if inline_styles
    end
  end

  doc
}

nest_styles = ->(css, selector) {
  require 'sass'
  Sass::Engine.new("#{selector} { #{css} }", syntax: :scss).to_css
}

file "build/full.html" => CHAPTERS do |task|
  require 'nokogiri'
  main_doc = nil
  main_body = nil
  chapter_ids = Hash.new { |h,k| h[k] = k.gsub(/[\W-]+/, '-') }
  translated_ids = Hash.new {|h,k| h[k] = {} }
  
  for html_file in CHAPTERS
    chapter = Nokogiri::HTML(File.read(html_file), nil, 'UTF-8')
    chapter_styles = chapter.css('style').map {|node| node.remove.text }
    chapter.css('link[rel=prefetch], link[rel=stylesheet], script, p#toc').each(&:remove)
    chapter.xpath('//a[starts-with(@href, "table-of-contents.html")]/..').each(&:remove)
    chapter.xpath('//div[@class="moneybags"]/../preceding-sibling::*[2]/following-sibling::node()').each(&:remove)
    
    # add space after <br> in titles to help Prince
    chapter.css((1..6).map {|n| "h#{n} br" }.join(', ')).each { |br| br.after(' ') }
    
    head = chapter.css('head').last
    body = chapter.at_css('body') || head.after('<body>').next_element
    
    found_body = false
    first_child = body.children.first
    head.children.each do |node|
      if found_body || body_element.(node)
        found_body = true
        first_child ? first_child.before(node) : body << node
      end
    end
    
    unless main_doc
      main_doc = Nokogiri::HTML::Document.new
      main_doc.root = main_doc.create_element 'html'
      main_body = main_doc.create_element 'body'
      main_doc.root << head << main_body
      
      require 'date'
      today = Date.today
      
      # Prince PDF metadata
      head << "\n" << (<<-HTML).lstrip
        <meta name="author" content="Mark Pilgrim">
        <meta name="subject" content="Hand-picked Selection of features from the HTML5 specification and other fine Standards.">
        <meta name="date" content="#{today.strftime('%Y-%m-%d')}">
      HTML
    end
    
    container = main_doc.create_element 'div'
    container['id'] = chapter_ids[html_file]
    add_class.(container, 'chapter')
    
    if chapter_styles.any?
      css = nest_styles.(chapter_styles.join("\n"), "##{container['id']}")
      container << "<style>\n#{css}</style>\n"
    end
    
    # ensure unique IDs
    dehtml5.(body).css('*[id]').each do |el|
      old_id = el['id']
      if main_body.at_xpath(%{id("#{old_id}")})
        chapter_id = container['id']
        new_id = "#{chapter_id}-#{old_id}"
        translated_ids[chapter_id][old_id] = new_id
        el['id'] = new_id
      end
    end
    
    container << body.children
    main_body.add_child container
  end
  
  # resolve cross-references, relative URLs
  main_body.css('a[href]').each do |link|
    href = link['href']
    new_href = nil
    
    if CHAPTERS.include? href
      new_href = "##{chapter_ids[href]}"
    elsif href =~ /#/
      page, ref = $`, $'
      if !ref.empty? and (page.empty? or CHAPTERS.include? page)
        chapter_id = page.empty? ? link.at_xpath('ancestor::div[@class="chapter"]/@id').text : chapter_ids[page]
        resolved_ref = translated_ids[chapter_id].fetch(ref, ref)
        new_href = "##{resolved_ref}"
      end
    end
    
    if !new_href and href !~ /^#/ and href !~ /^[\w-]+:/
      new_href = File.join(WEB_PREFIX, href)
    end
    
    link['href'] = new_href if new_href
  end
  
  syntax_highlighter.(main_body)
  
  main_body.xpath('//pre').each do |pre|
    lines = pre.text.split("\n")
    if lines.size > 10 # or lines.any? {|l| l.length > 80 }
      add_class.(pre, 'lots')
    end
  end
  
  FileUtils.mkdir_p File.dirname(task.name)
  File.open(task.name, 'w') { |f| f.write main_doc.to_html }
end

file "build/Dive into HTML5.pdf" => ["build/full.html", "highlight.css", "print.css"] do |task|
  system PRINCE, '--baseurl', '/', '--fileroot', Dir.pwd,
    '-s', 'screen.css', '-s', 'highlight.css', '-s', 'print.css',
    task.prerequisites.first, '-o', task.name
end

task :repdf do
  pdf = "build/Dive into HTML5.pdf"
  FileUtils.rm_f pdf
  Rake::Task[pdf].invoke
end

task :default => "build/Dive into HTML5.pdf" do |task|
  system 'open', task.prerequisites.first
end
