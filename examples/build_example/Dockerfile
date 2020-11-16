FROM harbor.iiis.co/library/ubuntu-pytorch:1.5.0
COPY . build                                         
RUN pip install -r build/requirements.txt           
RUN cd build/samplemod; pip install .
RUN rm -rf build                                      
