FROM harbor.iiis.co/library/ubuntu-pytorch:1.5.0
COPY . build                                         
RUN pip install -r build/requirements.txt && cd build/samplemod; pip install . && mkdir -p workspace && rm -rf build                                      
