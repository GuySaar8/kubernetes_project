FROM python:2.7-onbuild
ENV PYTHONPATH /usr/src/app
WORKDIR /app
ADD ./consumer-app/consumer.py /app
ADD requirements.txt /app
RUN python -m pip install -r requirements.txt
