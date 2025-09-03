ARG MELTANO_IMAGE=meltano/meltano:v2.19.1-python3.8
FROM $MELTANO_IMAGE

# Create a non-root user, create the /project folder, and set ownership
RUN useradd -m meltano_user && \
    mkdir -p /project && \
    chown -R meltano_user:meltano_user /project

# Switch to the non-root user
USER meltano_user

WORKDIR /project

# Install any additional requirements
COPY --chown=meltano_user:meltano_user requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy over Meltano project directory with appropriate ownership
COPY --chown=meltano_user:meltano_user . .
RUN meltano install

# Allow changes to containerized project files
ENV MELTANO_PROJECT_READONLY 0

ENTRYPOINT ["meltano"]
