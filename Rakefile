SLIDE_LEVEL = 2
THEME       = "yatil"

task(:default => :build)

task(:build) do
  `pandoc --slide-level=#{SLIDE_LEVEL} --variable=s5-url=ui/#{THEME} -i -s -t s5 slides.markdown -o index.html`
  `open index.html`
end
