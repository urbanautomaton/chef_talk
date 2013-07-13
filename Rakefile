task(:default => :build)

task(:build) do
  `pandoc --slide-level=2 --variable=s5-url=ui/yatil -s -t s5 slides.markdown -o slides.html`
  `open slides.html`
end
