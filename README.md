gcloud beta ai-platform versions delete
#!/bin/bash

PROJECT_ID=$(gcloud config get-value project)
SERVICES=$(gcloud beta run services list --format='value(SERVICE,REGION)')

MAX_RETRIES=3
RETRY_DELAY=5

function delete_revision_with_retry() {
  local REVISION=$1
  local SERVICE=$2
  local REGION=$3
  local RETRIES=$4

  if [[ $RETRIES -eq 0 ]]; then
    echo "[$REGION][$SERVICE] - Failed to delete revision $REVISION for Cloud Run $SERVICE in region $REGION after maximum retries. Moving on to the next revision."
    return
  fi

  echo "[$REGION][$SERVICE] - Deleting revision $REVISION for Cloud Run $SERVICE in region $REGION (Retry attempts left: $RETRIES)"
  gcloud beta run revisions delete $REVISION --region=$REGION --quiet

  if [[ $? -eq 0 ]]; then
    echo "[$REGION][$SERVICE] - Successfully deleted revision $REVISION for Cloud Run $SERVICE in region $REGION"
  else
    echo "[$REGION][$SERVICE] - Failed to delete revision $REVISION for Cloud Run $SERVICE in region $REGION. Retrying in $RETRY_DELAY seconds..."
    sleep $RETRY_DELAY
    delete_revision_with_retry $REVISION $SERVICE $REGION $((RETRIES - 1))
  fi
}

function delete_image_with_retry() {
  local SERVICE=$1
  local REGION=$2
  local IMAGE_URL=$3
  local RETRIES=$4

  if [[ $RETRIES -eq 0 ]]; then
    echo "[$REGION][$SERVICE] - Failed to delete image $IMAGE_URL after maximum retries."
    return
  fi

  echo "[$REGION][$SERVICE] - Deleting image $IMAGE_URL (Retry attempts left: $RETRIES)"
  gcloud container images delete $IMAGE_URL --force-delete-tags --quiet

  if [[ $? -eq 0 ]]; then
    echo "[$REGION][$SERVICE] - Successfully deleted image $IMAGE_URL"
  else
    echo "[$REGION][$SERVICE] - Failed to delete image $IMAGE_URL. Retrying in $RETRY_DELAY seconds..."
    sleep $RETRY_DELAY
    delete_image_with_retry $SERVICE $REGION $IMAGE_URL $((RETRIES - 1))
  fi
}

while read -r SERVICE REGION; do
  echo "Cloud Run: $SERVICE (Region: $REGION)"

  REVISIONS=$(gcloud beta run revisions list --service=$SERVICE --region=$REGION --format='value(metadata.name)' --sort-by='~metadata.creationTimestamp' --limit=9999)
  REVISIONS_TO_DELETE=$(echo "$REVISIONS" | tail -n +6)

  if [[ -z $REVISIONS_TO_DELETE ]]; then
    echo "[$REGION][$SERVICE] - Revisions to be deleted: none"
  else
    echo "[$REGION][$SERVICE] - Revisions to be deleted: $REVISIONS_TO_DELETE"
  fi

  for REVISION in $REVISIONS_TO_DELETE; do
    IMAGE_URL=$(gcloud beta run revisions describe $REVISION --region=$REGION --format='value(spec.containers[0].image)')
    delete_revision_with_retry $REVISION $SERVICE $REGION $MAX_RETRIES
    delete_image_with_retry $SERVICE $REGION $IMAGE_URL $MAX_RETRIES
  done

  echo ""
done <<< "$SERVICES"
